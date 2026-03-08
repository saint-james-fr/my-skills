# Geospatial Systems — Detailed Reference

## Geohash Encoding Step-by-Step

### Full Example: Encoding Google HQ (37.4220, -122.0841)

**Step 1 — Encode latitude 37.4220**:

```
Range [-90, 90] → 37.4220 is in [0, 90]    → bit 1
Range [0, 90]   → 37.4220 is in [0, 45]    → bit 0
Range [0, 45]   → 37.4220 is in [22.5, 45] → bit 1
Range [22.5, 45]→ 37.4220 is in [33.75, 45]→ bit 1
Range [33.75,45]→ 37.4220 is in [33.75,39.375]→ bit 0
...continue for desired precision
```

Latitude bits: `1 0 1 1 0 ...`

**Step 2 — Encode longitude -122.0841**:

```
Range [-180, 180] → -122.0841 is in [-180, 0]  → bit 0
Range [-180, 0]   → -122.0841 is in [-180, -90]→ bit 0
Range [-180, -90] → -122.0841 is in [-135, -90]→ bit 1
Range [-135, -90] → -122.0841 is in [-135,-112.5]→ bit 0
Range [-135,-112.5]→ -122.0841 is in [-135,-123.75]→ bit 0
...continue for desired precision
```

Longitude bits: `0 0 1 0 0 ...`

**Step 3 — Interleave (longitude first)**:

```
Longitude: 0  0  1  0  0  ...
Latitude:  1  0  1  1  0  ...
Combined:  01 00 11 01 00 ...
```

**Step 4 — Group into 5-bit chunks → base32**:

```
01001 → 9
10110 → q
01001 → 9
10000 → h
11011 → v
11010 → u
Result: 9q9hvu
```

### Base32 Character Map

```
0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
0  1  2  3  4  5  6  7  8  9  b  c  d  e  f  g

16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
h  j  k  m  n  p  q  r  s  t  u  v  w  x  y  z
```

Note: `a`, `i`, `l`, `o` are excluded to avoid confusion with digits.

### Neighbor Computation

Each geohash cell has exactly 8 neighbors. Computing them is O(1):

```
Given geohash "9q9hvu":
  N:  9q9hvv    NE: 9q9hvy    E: 9q9hvg
  SE: 9q9hvf    S:  9q9hvs    SW: 9q9hve
  W:  9q9hvt    NW: 9q9hvw
```

Algorithm: decode geohash to lat/lng, add/subtract one cell width, re-encode. Special handling at grid edges (wrap-around at meridian/equator).

### Geohash in SQL

```sql
-- Table structure (compound key, no locking needed)
CREATE TABLE geohash_index (
  geohash  VARCHAR(12) NOT NULL,
  business_id BIGINT NOT NULL,
  PRIMARY KEY (geohash, business_id)
);

-- Index on geohash for prefix queries
CREATE INDEX idx_geohash ON geohash_index (geohash);

-- Nearby search: current cell + 8 neighbors
SELECT business_id
FROM geohash_index
WHERE geohash IN ('9q9hvu', '9q9hvv', '9q9hvy', '9q9hvg',
                   '9q9hvf', '9q9hvs', '9q9hve', '9q9hvt', '9q9hvw');

-- Expanding search (remove last character for broader area)
SELECT business_id
FROM geohash_index
WHERE geohash LIKE '9q9hv%';
```

### Geohash in Redis

```
-- Store business locations
GEOADD businesses -122.0841 37.4220 "google-hq"
GEOADD businesses -122.1497 37.4847 "facebook-hq"

-- Find within radius
GEORADIUS businesses -122.0841 37.4220 5 km WITHCOORD WITHDIST COUNT 20 ASC

-- Get geohash
GEOHASH businesses "google-hq"
-- Returns: "9q9hvu0..."

-- Alternative: manual key-value approach for custom indexing
SET geohash:9q9hvu "[343, 347, 892]"  -- list of business IDs
```

## Quadtree Memory Calculation

### Detailed Breakdown (200 million businesses)

**Leaf node structure**:

```
┌──────────────────────────────────────────┐
│ Leaf Node (832 bytes)                    │
├──────────────────────────────────────────┤
│ top_left_lat     : 8 bytes (double)     │
│ top_left_lng     : 8 bytes (double)     │
│ bottom_right_lat : 8 bytes (double)     │
│ bottom_right_lng : 8 bytes (double)     │
│ business_ids[100]: 800 bytes (8B × 100) │
└──────────────────────────────────────────┘
```

**Internal node structure**:

```
┌──────────────────────────────────────────┐
│ Internal Node (64 bytes)                 │
├──────────────────────────────────────────┤
│ top_left_lat     : 8 bytes (double)     │
│ top_left_lng     : 8 bytes (double)     │
│ bottom_right_lat : 8 bytes (double)     │
│ bottom_right_lng : 8 bytes (double)     │
│ child_NW         : 8 bytes (pointer)    │
│ child_NE         : 8 bytes (pointer)    │
│ child_SW         : 8 bytes (pointer)    │
│ child_SE         : 8 bytes (pointer)    │
└──────────────────────────────────────────┘
```

**Calculation**:

```
Given: 200M businesses, max 100 per leaf

Leaf nodes    = 200,000,000 / 100 = 2,000,000
Internal nodes = leaf_nodes / 3   = 666,667

  (In a quadtree, #internal ≈ #leaf / 3 because each internal
   node has 4 children, and the sum of a geometric series
   1 + 1/4 + 1/16 + ... converges to 4/3 of leaf count.
   Internal = leaf × 1/3.)

Leaf memory     = 2,000,000 × 832 bytes   = 1,664,000,000 bytes ≈ 1.66 GB
Internal memory = 666,667   × 64 bytes    =    42,666,688 bytes ≈ 0.04 GB

Total = 1.66 + 0.04 = 1.70 GB ≈ 1.71 GB
```

### Build Time

```
Time complexity: O((n/100) × log(n/100))
  where n = 200,000,000

  = O(2,000,000 × log(2,000,000))
  = O(2,000,000 × 21)
  = O(42,000,000) operations

Estimate: a few minutes on modern hardware
```

### Quadtree Pseudocode (Complete)

```
class QuadTreeNode:
    bounds: BoundingBox      # (top_left, bottom_right)
    children: [4]QuadTreeNode  # NW, NE, SW, SE (null for leaf)
    business_ids: List[int]    # populated only for leaves

def build(node: QuadTreeNode, all_businesses: List):
    businesses_in_node = filter(all_businesses, node.bounds)

    if len(businesses_in_node) <= MAX_PER_LEAF:   # e.g. 100
        node.business_ids = [b.id for b in businesses_in_node]
        return

    mid_lat = (node.bounds.top + node.bounds.bottom) / 2
    mid_lng = (node.bounds.left + node.bounds.right) / 2

    node.children[NW] = QuadTreeNode(bounds=(top, left, mid_lat, mid_lng))
    node.children[NE] = QuadTreeNode(bounds=(top, mid_lng, mid_lat, right))
    node.children[SW] = QuadTreeNode(bounds=(mid_lat, left, bottom, mid_lng))
    node.children[SE] = QuadTreeNode(bounds=(mid_lat, mid_lng, bottom, right))

    for child in node.children:
        build(child, businesses_in_node)

def search(node: QuadTreeNode, lat: float, lng: float, k: int) -> List[int]:
    if node.is_leaf():
        return node.business_ids

    for child in node.children:
        if child.bounds.contains(lat, lng):
            results = search(child, lat, lng, k)
            if len(results) >= k:
                return results
            # expand to siblings/neighbors for more results
            return results + search_neighbors(node, child, k - len(results))
```

## Google S2 Cell Hierarchy

### How S2 Works

1. **Sphere → Cube**: project the Earth's surface onto 6 faces of a surrounding cube
2. **Cube face → Hilbert curve**: map each face to a 1D space using a Hilbert curve (space-filling curve that preserves locality)
3. **Cell IDs**: each point on the Hilbert curve gets a 64-bit cell ID encoding face + position + level

### S2 Cell Levels

```
Level 0:  6 cells (cube faces), ~85M km² each
Level 1:  24 cells, ~21M km²
Level 5:  6,144 cells, ~14,000 km²
Level 10: ~6M cells, ~13.6 km²
Level 15: ~6B cells, ~13.3 m²
Level 20: ~6T cells, ~0.013 m²
Level 30: finest, ~0.74 cm²

Total: 31 levels (0–30)
```

### Key Property: Hilbert Curve Locality

```
2D Space:               1D Hilbert Index:
┌───┬───┐
│ 1 │ 2 │              1 → 2 → 3 → 4 → 5 → 6 → 7 → ...
├───┼───┤
│ 4 │ 3 │              Points close in 2D remain close in 1D
├───┼───┤              (unlike Z-order/geohash which has jumps)
│ 5 │ 8 │
├───┼───┤
│ 6 │ 7 │
└───┴───┘
```

### Region Cover Algorithm

Unlike Geohash (fixed precision), S2 can cover arbitrary regions with mixed-level cells:

```
Parameters:
  min_level: 8     # minimum cell granularity
  max_level: 15    # maximum cell granularity
  max_cells: 20    # max number of cells to use

Input: a circular region (center + radius)
Output: 12–20 cells of varying sizes that tightly cover the region

Advantage: fewer false positives than geohash (tighter boundary fit)
```

### S2 vs Geohash for Geofencing

```
Geohash (fixed grid):           S2 (adaptive cells):
┌──┬──┬──┬──┐                   ┌──────┬─────┐
│  │  │  │  │                   │      │  ┌──┤
├──┼──┼──┼──┤                   │      │  └──┤
│  │//│//│  │ ← Many cells      │  ┌───┤     │ ← Fewer cells,
├──┼──┼──┼──┤   needed to       │  │///│     │   better fit
│  │//│//│  │   approximate     │  └───┤     │
├──┼──┼──┼──┤   a circle        └──────┴─────┘
│  │  │  │  │
└──┴──┴──┴──┘
```

## Routing Graph Construction

### From Raw Data to Routing Tiles

```
Raw Road Data (TB)
  │
  ▼
┌───────────────────────┐
│ Routing Tile Processor │  ← Periodic offline batch job
│ (extracts road graph)  │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────────────┐
│ Object Storage (S3)           │
│                               │
│ /routing-tiles/               │
│   /level-1/  (highways)       │
│     /9q9h.bin                 │
│     /9q9j.bin                 │
│   /level-2/  (arterials)      │
│     /9q9hv.bin                │
│     /9q9hw.bin                │
│   /level-3/  (local streets)  │
│     /9q9hvu.bin               │
│     /9q9hvv.bin               │
└───────────────────────────────┘
```

### Routing Tile Binary Format

Each tile is serialized as an adjacency list:

```
RoutingTile {
  tile_id: string (geohash)
  level: int (1=highway, 2=arterial, 3=local)
  nodes: [
    { id: int, lat: double, lng: double }
  ]
  edges: [
    { from: int, to: int, distance_m: int, speed_limit: int,
      road_name: string, is_one_way: bool }
  ]
  connections: [
    { node_id: int, target_tile_id: string, target_node_id: int,
      target_level: int }  # cross-tile and cross-level references
  ]
}
```

### Path Finding Across Tiles

```
Origin tile (local) ──edge──▶ Neighboring tile (local)
       │                              │
       │ cross-level edge             │
       ▼                              ▼
Highway tile ────────edge────────▶ Highway tile
       │                              │
       │ cross-level edge             │
       ▼                              ▼
Local tile (near dest) ──edge──▶ Destination tile (local)
```

**Algorithm flow**:
1. Load origin routing tile from S3/cache
2. Run A* expanding outward, loading neighbor tiles on demand
3. When entering a highway on-ramp, switch to highway-level tiles (fewer nodes, faster traversal)
4. Near destination, switch back to local-level tiles
5. Return top-k shortest paths

## Map Tile Rendering Pipeline

### Pre-computation Process

```
Road Data + Satellite Imagery + Labels + POI Data
        │
        ▼
┌────────────────────┐
│  Tile Renderer     │  ← Offline batch process
│  (per zoom level)  │
└────────┬───────────┘
         │
         ▼
┌──────────────────────────────┐
│ For each zoom level 0–21:    │
│   For each grid cell:        │
│     1. Clip road/feature     │
│        data to cell bounds   │
│     2. Apply styling rules   │
│        (road width, colors,  │
│         labels, POI icons)   │
│     3. Render 256×256 PNG    │
│     4. Store: s3://tiles/    │
│        {zoom}/{geohash}.png  │
└──────────────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ CDN Distribution             │
│ URL: cdn.map.com/tiles/      │
│      {zoom}/{geohash}.png    │
│ Headers: Cache-Control:      │
│   public, max-age=31536000   │
└──────────────────────────────┘
```

### Storage Math

```
Zoom 21: 4.4T tiles × 100KB = 440 PB (raw)
  ~90% Earth is ocean/desert → 80-90% compression
  Effective: ~44-88 PB → round to 50 PB

Lower zoom levels (geometric series):
  50 + 50/4 + 50/16 + 50/64 + ... = ~67 PB

Total: ~100 PB across all zoom levels
```

### Client-Side Tile Fetching

```
1. User viewport → determine visible area (lat/lng bounds)
2. Current zoom level → select tile set
3. For each visible grid cell:
   a. Compute geohash from (lat, lng, zoom)
   b. Check local tile cache
   c. If miss → fetch from CDN: GET /tiles/{zoom}/{geohash}.png
4. Stitch tiles into mosaic for rendering
5. Pre-fetch adjacent tiles for smooth panning
```

## Database Schemas for Geo Data

### Business Table

```sql
CREATE TABLE business (
  business_id   BIGINT PRIMARY KEY,
  name          VARCHAR(255) NOT NULL,
  address       VARCHAR(500),
  city          VARCHAR(100),
  state         VARCHAR(50),
  country       VARCHAR(50),
  latitude      DECIMAL(10, 7) NOT NULL,
  longitude     DECIMAL(10, 7) NOT NULL,
  category      VARCHAR(50),
  rating        DECIMAL(2, 1),
  is_open       BOOLEAN DEFAULT true,
  created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_business_location ON business (latitude, longitude);
```

### Geospatial Index Table

```sql
-- Option 1: compound key (recommended)
CREATE TABLE geospatial_index (
  geohash      VARCHAR(12) NOT NULL,
  business_id  BIGINT NOT NULL,
  PRIMARY KEY (geohash, business_id),
  FOREIGN KEY (business_id) REFERENCES business(business_id)
);

-- Enables efficient prefix queries
CREATE INDEX idx_geohash ON geospatial_index (geohash);
```

### Location History (Cassandra)

```cql
CREATE TABLE location_history (
  user_id     UUID,
  timestamp   BIGINT,
  lat         DOUBLE,
  lng         DOUBLE,
  user_mode   TEXT,        -- active, inactive
  nav_mode    TEXT,        -- driving, walking, cycling
  PRIMARY KEY (user_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

Partition key = `user_id` ensures even distribution. Clustering by `timestamp DESC` makes latest-location queries fast.

### Redis Cache Patterns

```
# Geohash → business IDs (with 24h TTL)
SET  geohash:4:9q9h    "[101, 205, 309, ...]"  EX 86400
SET  geohash:5:9q9hv   "[101, 205]"            EX 86400
SET  geohash:6:9q9hvu  "[101]"                 EX 86400

# Business info cache
HSET business:101 name "Coffee Shop" lat 37.422 lng -122.084 rating 4.5

# User location (with TTL for activity detection)
SET  user_loc:user_42  "{lat:37.42, lng:-122.08, ts:1635740977}"  EX 600
```

### Nearby Friends — Redis Pub/Sub Pattern

```
# Channel per user
SUBSCRIBE user:42:location    # friend subscribes to user 42's updates
PUBLISH   user:42:location    "{lat:37.42, lng:-122.08, ts:...}"

# Memory: 100M channels × 100 friends × 20B pointers ≈ 200 GB
# CPU bottleneck: ~13M location pushes/sec → shard across ~130 Redis servers
# Shard using consistent hashing on user_id
```

## Complete System Design Checklist

### Proximity Service Interview Checklist

```
□ Clarify scope: nearby search vs CRUD vs real-time updates
□ APIs: GET /search/nearby (lat, lng, radius), CRUD for businesses
□ Discuss geospatial indexing: geohash vs quadtree vs S2
□ Explain chosen approach with boundary handling
□ Data model: business table + geo index table (compound key)
□ High-level arch: LBS (stateless, read) + Business Service (CRUD)
□ Database: MySQL primary + read replicas
□ Caching: Redis for geohash→business_ids + business objects
□ Scaling: replicas for geo index, shard business table by ID
□ Multi-region deployment for latency + privacy compliance
```

### Google Maps Interview Checklist

```
□ Scope: location updates, navigation/ETA, map rendering
□ Map 101: tiles, zoom levels, geohash for tile addressing
□ Location service: batch updates, Cassandra, Kafka streaming
□ Navigation: geocoding → shortest path → ETA → ranking
□ Routing tiles: hierarchical (local/arterial/highway), S3 storage
□ Map rendering: pre-computed tiles, CDN, vector tile optimization
□ Adaptive ETA: track active sessions, reroute on traffic changes
□ Delivery: WebSocket for server-push during navigation
□ Numbers: 1B DAU, 200K location QPS, 36K peak nav QPS, ~100 PB tiles
```
