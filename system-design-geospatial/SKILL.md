---
name: system-design-geospatial
description: Geospatial system design — proximity services, geohash, quadtree, Google S2, map rendering, routing algorithms, and location-based search. Use when designing location-based services, implementing nearby search, building mapping systems, or choosing geospatial indexing strategies.
---

# Geospatial System Design

## Core Idea

All geospatial indexing shares one principle: **divide the map into smaller areas and build indexes for fast search**, converting 2D coordinates into 1D-searchable keys.

Two families of approach:

| Family | Algorithms | Storage |
|--------|-----------|---------|
| **Hash-based** | Even grid, **Geohash**, Cartesian tiers | DB column / Redis |
| **Tree-based** | **Quadtree**, **Google S2**, R-tree | In-memory structure |

## 1. Geospatial Indexing — Comparison

| | Geohash | Quadtree | Google S2 |
|---|---------|----------|-----------|
| **How it works** | Recursively bisects lat/lng → interleaved bits → base32 string | Recursively subdivides 2D space into 4 quadrants until threshold (e.g. ≤100 items per cell) | Maps sphere onto cube faces → Hilbert curve → 1D cell IDs |
| **Index type** | DB column / Redis sorted set | In-memory tree on each server | In-memory; Hilbert curve cells |
| **Grid size** | Fixed per precision level | Adaptive — denser areas get smaller cells | Adaptive — Region Cover algo picks variable-level cells |
| **Pros** | Simple, easy updates, range queries via prefix | k-nearest natural, adaptive density, ~1.7 GB for 200M POIs | Geofencing, arbitrary region cover, fine-grained |
| **Cons** | Boundary issues, fixed grid size | Complex updates (tree traversal + locking), slow startup | Complex to explain, harder to implement |
| **Used by** | Bing, Redis, MongoDB, Lyft, Elasticsearch | Yext, Elasticsearch | Google Maps, Tinder |

**Interview tip**: choose Geohash or Quadtree. S2 is hard to explain clearly under time pressure.

## 2. Geohash

### Encoding Process

```
lat/lng → recursive binary subdivision → interleave bits → base32

Example (Google HQ, precision 6):
  Binary:  1001 10110 01001 10000 11011 11010
  Base32:  9q9hvu
```

**Step-by-step**:
1. Divide planet into 4 quadrants (prime meridian + equator)
2. Latitude [-90,0] → 0, [0,90] → 1; Longitude [-180,0] → 0, [0,180] → 1
3. Alternate longitude/latitude bits, subdivide recursively
4. Group into 5-bit chunks → base32 characters

### Precision Levels

| Length | Grid size | Use case |
|--------|-----------|----------|
| 1 | 5,009 km × 4,993 km | Planet |
| 2 | 1,252 km × 624 km | Continent |
| 3 | 156 km × 156 km | Large region |
| **4** | **39 km × 19.5 km** | 5–20 km radius search |
| **5** | **4.9 km × 4.9 km** | 1–2 km radius search |
| **6** | **1.2 km × 609 m** | 0.5 km radius search |
| 7 | 153 m × 152 m | Block level |
| 8 | 38 m × 19 m | Building |
| 9–12 | < 5 m | Precision GPS |

### Radius → Geohash Length

| Radius | Geohash length |
|--------|---------------|
| 0.5 km | 6 |
| 1 km | 5 |
| 2 km | 5 |
| 5 km | 4 |
| 20 km | 4 |

### Boundary Issues

**Issue 1 — Close but no shared prefix**: two locations on opposite sides of equator/prime meridian share no prefix despite being close (e.g. La Roche-Chalais `u000` vs Pomerol `ezzz` — 30 km apart).

**Issue 2 — Long shared prefix but different cells**: positions in adjacent cells can share a long prefix yet belong to different geohash regions.

**Solution**: always query the current cell **+ 8 neighbors**. Neighbor geohashes are computed in O(1).

```
┌───────┬───────┬───────┐
│ NW    │  N    │  NE   │
├───────┼───────┼───────┤
│  W    │ SELF  │  E    │
├───────┼───────┼───────┤
│ SW    │  S    │  SE   │
└───────┴───────┴───────┘
  Query: WHERE geohash IN (self, N, NE, E, SE, S, SW, W, NW)
```

### Geohash Range Queries

```sql
SELECT business_id FROM geohash_index
WHERE geohash LIKE '{:geohash_prefix}%'
```

**Not enough results?** Remove last character → expand grid → repeat.

```
9q9hvu → 9q9hv → 9q9h  (each step: ~4× larger area)
```

## 3. Quadtree

### How It Works

```
                    ┌─────────────┐
                    │   World     │
                    │  200M POIs  │
                    └──────┬──────┘
               ┌─────┬────┴────┬─────┐
               │ NW  │ NE     │ SW  │ SE  │
               │>100 │ ≤100   │>100 │>100 │
               │split│ LEAF   │split│split│
               ▼     ▼        ▼     ▼
             split  done    split  split
              ...            ...    ...
```

**Rule**: subdivide until each leaf has ≤ threshold items (e.g. 100 businesses).

```
buildQuadtree(node):
  if count(node) > 100:
    node.subdivide()
    for child in node.children:
      buildQuadtree(child)
```

### Node Data

| Node type | Data | Size |
|-----------|------|------|
| **Leaf** | Bounding box (4 × 8B) + up to 100 business IDs (100 × 8B) | 832 bytes |
| **Internal** | Bounding box (4 × 8B) + 4 child pointers (4 × 8B) | 64 bytes |

### Memory Estimation (200M businesses)

| Metric | Value |
|--------|-------|
| Leaf nodes | 200M / 100 = 2M |
| Internal nodes | 2M × 1/3 ≈ 0.67M |
| Leaf memory | 2M × 832B = 1.66 GB |
| Internal memory | 0.67M × 64B = 0.04 GB |
| **Total** | **~1.71 GB** |

Fits easily in one server's memory. Replicate for read throughput.

### Search Process

1. Build quadtree in memory at server startup
2. Traverse from root → find leaf containing search origin
3. If leaf has enough results, return
4. Otherwise, add neighbors until enough results

### Operational Considerations

- **Startup**: few minutes to build with 200M POIs — roll out incrementally (avoid brownout)
- **Updates**: easiest = nightly rebuild; harder = live update with locking
- **Blue/green deploy**: possible but all servers fetching 200M records simultaneously strains DB
- **Rebalancing**: needed when leaf overflows — over-allocate ranges as mitigation

### Geohash vs Quadtree Decision

| Criterion | Geohash | Quadtree |
|-----------|---------|----------|
| Implementation | Simple — no tree | Harder — tree + build time |
| k-nearest query | Needs workaround | Natural fit |
| Adaptive density | No (fixed grid) | Yes |
| Index updates | O(1) remove row | O(log n) tree traversal |
| Concurrency | No locking needed | Locking required for writes |
| Storage | DB rows | In-memory per server |

## 4. Proximity Service Design

### Architecture

```
  Client
    │
    ▼
┌──────────┐
│   LB     │
└────┬─────┘
     ├──────────────────────┐
     ▼                      ▼
┌──────────┐         ┌──────────────┐
│   LBS    │         │  Business    │
│ (read)   │         │  Service     │
│ stateless│         │ (CRUD)       │
└────┬─────┘         └──────┬───────┘
     │                      │
     ▼                      ▼
┌────────────────────────────────┐
│     Database Cluster           │
│  Primary (write) + Replicas   │
│  ┌────────┐  ┌─────────────┐  │
│  │Business│  │Geospatial   │  │
│  │ Table  │  │Index Table  │  │
│  └────────┘  └─────────────┘  │
└────────────────────────────────┘
```

### APIs

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/search/nearby?lat=X&lng=Y&radius=Z` | GET | Find nearby businesses |
| `/v1/businesses/{id}` | GET | Business detail |
| `/v1/businesses` | POST | Add business |
| `/v1/businesses/{id}` | PUT | Update business |
| `/v1/businesses/{id}` | DELETE | Remove business |

### Key Design Decisions

**Read-heavy workload**: search QPS ~5,000 (100M DAU × 5 queries / 86,400s). Write volume is low (business CRUD is infrequent). MySQL + read replicas is a good fit.

**Geospatial index table** — prefer compound key `(geohash, business_id)` over JSON array per geohash:
- No locking needed for add/remove
- Simple INSERT/DELETE vs scan-and-update

**Scaling the geo index**: full dataset ~1.7 GB → fits in working set. Scale with read replicas, not sharding. Sharding adds complexity with no strong technical reason at this data size.

### Caching Layer

| Cache | Key | Value |
|-------|-----|-------|
| Geohash cache | geohash string | List of business_id |
| Business cache | business_id | Business object |

Cache all 3 active precisions (length 4, 5, 6):
- 200M businesses × 3 precisions × 8 bytes = ~5 GB
- Fits in a single Redis instance; deploy globally for low latency

**Cache key**: use geohash (not raw lat/lng) — small location changes still map to same geohash.

### Query Flow

1. Client sends `(lat, lng, radius)` to LBS
2. LBS maps radius → geohash length (e.g. 500m → length 6)
3. Compute geohash + 8 neighbors → 9 geohash prefixes
4. For each prefix, fetch business IDs from Redis (parallel)
5. Hydrate business objects from Business cache
6. Compute distances, rank, return top results

## 5. Google Maps Design

### Map Rendering with Tiles

**Core concept**: pre-generate static 256×256 px PNG tiles at each zoom level. Serve via CDN.

| Zoom | Tiles | Coverage per tile |
|------|-------|-------------------|
| 0 | 1 | Entire world |
| 1 | 4 | Hemisphere |
| 7 | 16,384 | ~City |
| 14 | 268M | ~Block |
| 21 | 4.4 trillion | ~Meter |

Each zoom level doubles tiles in each direction (4× total). ~90% of Earth is uninhabited → ~80-90% image compression. Total storage estimate: **~100 PB**.

### Tile URL Resolution

Client computes geohash from `(lat, lng, zoom)` → fetches tile:

```
https://cdn.mapprovider.com/tiles/{geohash}.png
```

Alternative: server-side tile URL service for operational flexibility (avoids hard-coded client algorithm).

### CDN Serving

- 5B navigation minutes/day × 1.25 MB/min = 6.25 PB/day
- With 200 POPs → ~300 MB/s per POP
- Static tiles are highly cacheable

### Vector Tiles Optimization

Vector tiles (paths + polygons) instead of raster PNGs:
- Much better compression → lower bandwidth
- Smooth zooming (no pixelation between levels)
- Client renders via WebGL

## 6. Routing & Navigation

### Road Graph

Roads = edges, intersections = nodes. Full world graph is too large for any pathfinding algorithm.

### Routing Tiles

Break road network into small grids (routing tiles). Each tile contains:
- Graph nodes (intersections) and edges (roads) within the area
- References to connected neighboring tiles

Stored in object storage (S3), aggressively cached on routing servers.

### Hierarchical Routing Tiles

| Level | Contains | Tile size |
|-------|----------|-----------|
| **Local** | Streets, local roads | Small |
| **Arterial** | District connectors | Medium |
| **Highway** | Highways, interstates | Large |

Cross-level edges connect local streets to highway on-ramps. A freeway entrance from street A creates an edge from the local tile node to the highway tile node.

### Shortest Path Service

1. Convert origin/destination lat/lng → geohash → load start/end routing tiles
2. Run A* variant, expanding search by hydrating neighboring tiles on demand
3. Cross resolution levels as needed (local → highway → local)
4. Return top-k shortest paths (structure-only, ignoring traffic)

### ETA Service

- Takes shortest paths + current traffic + historical data
- ML model predicts travel time including future traffic conditions
- Ranker applies user filters (avoid tolls, avoid highways) → returns ranked routes

### Adaptive Rerouting

- Track active navigation sessions: store `user → [routing_tile_1, tile_2, ...]`
- On traffic incident in tile X: find affected users
- Optimization: store hierarchical tiles per user `(current_tile, super_tile, super_super_tile)` — quickly filter unaffected users
- Recalculate ETA, push update via WebSocket

## 7. Location Service

### Periodic Updates

```
Client → batch GPS readings (every 1s) → send batch every 15s → Server

Write QPS: 1B DAU × 5 nav sessions / 7 days ÷ 86,400s ÷ 15s batch = ~200K
Peak QPS: ~1M
```

### Storage

| Store | Purpose | Technology |
|-------|---------|------------|
| Location cache | Current user position (TTL-based) | Redis |
| Location history | Historical positions | Cassandra (write-optimized) |
| Stream | Real-time processing | Kafka |

**Schema** (Cassandra):

| Column | Type | Role |
|--------|------|------|
| user_id | UUID | Partition key |
| timestamp | BIGINT | Clustering key |
| lat | DOUBLE | |
| lng | DOUBLE | |
| user_mode | TEXT | active/inactive |
| nav_mode | TEXT | driving/walking |

Location stream feeds into: live traffic service, routing tile updates, road detection, analytics.

## 8. Operational Considerations

### Multi-Region Deployment

- Deploy LBS close to users (US-West, Europe, Asia)
- Reduce latency, spread traffic across dense populations
- Comply with data privacy laws (GDPR, CCPA) — keep data local

### Database Strategy

| Data | DB | Scaling |
|------|----|---------|
| Business data | MySQL | Shard by business_id |
| Geo index | MySQL | Read replicas (data is small) |
| Location history | Cassandra | Shard by user_id |
| Geocoding | Redis | Key-value lookup |
| Routing tiles | S3 + local cache | Object storage |
| Map tiles | S3 + CDN | Static asset serving |

### Key Numbers

| Metric | Value |
|--------|-------|
| Nearby search QPS | ~5,000 (peak: 25K) |
| Quadtree memory (200M POIs) | ~1.71 GB |
| Geohash Redis cache (3 precisions) | ~5 GB |
| Map tile storage (all zooms) | ~100 PB |
| Location update QPS | ~200K (peak: 1M) |
| Navigation QPS | ~7,200 (peak: 36K) |

## Detailed Reference

For step-by-step encoding examples, memory calculations, S2 cell hierarchy, routing graph construction, and database schemas, see [geospatial-detail.md](references/geospatial-detail.md).
