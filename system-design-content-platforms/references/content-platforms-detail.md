# Content Platforms — Detailed Reference

Extended reference for URL shortener, web crawler, news feed, search autocomplete, and video streaming system designs.

---

## 1. URL Shortener — Hash Algorithms Deep Dive

### Algorithm Comparison

| Algorithm | Type | Output Length | Collision Risk | Speed | Notes |
|-----------|------|-------------|----------------|-------|-------|
| CRC32 | Checksum | 32 bits (8 hex) | High at scale | Very fast | Not cryptographic; truncate to 7 still needs collision handling |
| MurmurHash3 | Non-crypto hash | 32 or 128 bits | Low | Very fast | Good distribution; commonly used in hash tables |
| MD5 | Cryptographic | 128 bits (32 hex) | Extremely low | Moderate | Overkill for URL shortening; truncation defeats purpose |
| SHA-1 | Cryptographic | 160 bits (40 hex) | Extremely low | Slower | Deprecated for security; still usable for non-security hashing |

### Hash + Collision Resolution Flow

```
Input: longURL
  │
  ├─ hash(longURL) → take first 7 chars → shortURL candidate
  │
  ├─ Check DB/Bloom filter: shortURL exists?
  │     ├─ No  → save and return shortURL
  │     └─ Yes → append predefined string to longURL
  │              └─ Rehash → new candidate → check again (recursive)
  │
  └─ Bloom filter optimization: probabilistic check before DB query
     - False positives possible (unnecessary DB check)
     - False negatives impossible (never miss a collision)
```

### Base-62 Conversion Walkthrough

```
ID = 2009215674938 (from distributed ID generator)

Base-62 character set:
  0-9  → 0-9
  a-z  → 10-35
  A-Z  → 36-61

2009215674938 ÷ 62 = 32406704434 remainder 30 → 'u'
32406704434  ÷ 62 = 522688781  remainder 12 → 'c'
522688781   ÷ 62 = 8430464   remainder 13 → 'd'
8430464    ÷ 62 = 135975   remainder 14 → 'e'
135975     ÷ 62 = 2193     remainder 9  → '9'
2193       ÷ 62 = 35       remainder 23 → 'n'
35         ÷ 62 = 0        remainder 35 → 'z'

Read remainders bottom-up: "zn9edcu"

Short URL: https://tinyurl.com/zn9edcu
```

### Back-of-Envelope Estimates

| Metric | Value |
|--------|-------|
| Write QPS | 100M URLs/day ÷ 86,400 = ~1,160/s |
| Read QPS (10:1 ratio) | ~11,600/s |
| 10-year storage | 365 billion records |
| Storage size (avg 100 bytes/URL) | ~365 TB |
| Hash length needed | 7 chars (62^7 = 3.5T > 365B) |

### Data Model

```
Table: url_mapping
├── id          BIGINT  PRIMARY KEY   (unique ID from generator)
├── short_url   VARCHAR(7)            (base-62 encoded)
└── long_url    TEXT                   (original URL)

Index: short_url (for redirect lookups)
Cache: Redis <short_url → long_url> (read-heavy)
```

---

## 2. Web Crawler — Scalability & Estimates

### Back-of-Envelope Estimates

| Metric | Value |
|--------|-------|
| Pages/month | 1 billion |
| QPS | ~400 pages/s |
| Peak QPS | ~800 pages/s |
| Avg page size | 500 KB |
| Monthly storage | 500 TB |
| 5-year storage | 30 PB |

### URL Frontier — Complete Architecture

```
                    ┌──────────────┐
    URLs in ──────→ │  Prioritizer │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           ┌─────┐     ┌─────┐     ┌─────┐
           │ f1  │     │ f2  │     │ fn  │    ← Front queues (priority)
           │(high│     │(med)│     │(low)│
           └──┬──┘     └──┬──┘     └──┬──┘
              └────────────┼────────────┘
                           │
                    ┌──────▼───────┐
                    │Queue Selector│ ← Biased random pick
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Queue Router │ ← Routes to per-host queue
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           ┌─────┐     ┌─────┐     ┌─────┐
           │ b1  │     │ b2  │     │ bn  │    ← Back queues (politeness)
           │wiki │     │apple│     │nike │       One per host
           └──┬──┘     └──┬──┘     └──┬──┘
              ▼            ▼            ▼
           Worker 1    Worker 2    Worker N       One per queue, with delay
```

### Content Deduplication — SimHash

Standard hash (MD5, SHA) produces completely different output for minor changes. SimHash produces similar hashes for similar documents.

**SimHash algorithm:**
1. Tokenize document into features (words, shingles)
2. Hash each feature to a bit vector
3. Weight each bit position across all features
4. Produce final fingerprint from weighted sums
5. Compare fingerprints: Hamming distance < threshold → duplicate

| Technique | Exact Match | Near-Duplicate | Speed |
|-----------|-------------|---------------|-------|
| MD5/SHA hash | Yes | No | Fast |
| SimHash | No | Yes | Fast |
| Character comparison | Yes | Yes | Very slow |
| Jaccard similarity (shingles) | No | Yes | Moderate |

### Robots.txt Parsing

```
# Example robots.txt
User-agent: *
Disallow: /admin/
Disallow: /private/
Crawl-delay: 10

User-agent: Googlebot
Disallow: /creatorhub/*
Allow: /public/
```

Rules: check specific user-agent first, then wildcard. Cache robots.txt per domain with TTL (usually 24h).

### Extensibility Pattern

The crawler pipeline is designed as a chain of pluggable modules:

```
URL Frontier → Downloader → Parser → [Plugin modules] → URL Extractor → Filter → URL Frontier

Pluggable modules:
  ├── PNG Downloader (image crawling)
  ├── PDF Parser (document indexing)
  ├── Web Monitor (copyright detection)
  └── Custom extractors (structured data)
```

---

## 3. News Feed — Detailed Fan-out Flow

### Fan-out on Write — Step-by-Step

```
1. User publishes post: POST /v1/me/feed

2. Web server:
   ├── Authenticate (auth_token)
   ├── Rate-limit check
   └── Forward to Post Service

3. Post Service:
   ├── Write post to Post DB
   ├── Write post to Post Cache
   └── Trigger Fanout Service

4. Fanout Service:
   ├── Query Graph DB → get friend IDs (up to 5000)
   ├── Query User Cache → get friend settings
   ├── Filter:
   │   ├── Remove muted friends
   │   ├── Apply privacy settings
   │   └── Apply content filters
   ├── Push (post_id, friend_id) pairs to Message Queue
   └── Workers consume queue:
       └── For each friend:
           └── LPUSH post_id to friend's News Feed Cache list
               (capped at configurable limit, e.g., 500 posts)

5. Notification Service:
   └── Send push notification to friend's devices
```

### Fan-out on Read — Step-by-Step (for celebrities)

```
1. User requests feed: GET /v1/me/feed

2. News Feed Service:
   ├── Fetch pre-computed feed IDs from News Feed Cache (fan-out on write posts)
   ├── Identify celebrity accounts user follows
   ├── For each celebrity:
   │   └── Fetch latest N posts from celebrity's Post Cache
   ├── Merge + sort all posts by timestamp (reverse chronological)
   ├── Take top K posts
   └── Hydrate with full data from User Cache + Post Cache

3. Return fully hydrated JSON feed to client
```

### Cache Sizing Estimates

| Cache Layer | Estimate | Reasoning |
|-------------|----------|-----------|
| News Feed (per user) | 500 post IDs × 8 bytes = 4 KB | Most users view < 500 posts |
| 10M DAU total | ~40 GB | Fits in a Redis cluster |
| Content Cache | Hot posts only (~20% of all posts) | Follows Pareto distribution |
| Social Graph | Adjacency lists; 5000 friends × 8 bytes × 10M users | ~400 GB partitioned |

### Ranking Considerations

Beyond reverse chronological, production feeds typically use ML-based ranking:

| Signal | Weight Factor |
|--------|--------------|
| Recency | Time decay function |
| Relationship strength | Interaction frequency with poster |
| Content type preference | User's historical engagement by type |
| Engagement velocity | How fast the post is getting likes/comments |
| Diversity | Avoid showing too many posts from same source |

---

## 4. Search Autocomplete — Trie Implementation Detail

### Basic Trie Node Structure

```
TrieNode {
  children: Map<char, TrieNode>  // up to 26 for lowercase English
  isEndOfWord: boolean
  frequency: number              // query frequency for this word
  topK: List<(query, freq)>      // cached top-K queries from this prefix
}
```

### Space Optimization Techniques

| Technique | How | Space Savings |
|-----------|-----|--------------|
| **Compressed trie (Patricia trie)** | Merge single-child chains into one node ("app" instead of a→p→p) | Significant for sparse tries |
| **Array vs Map for children** | Use fixed 26-element array if most nodes have many children; Map if sparse | Depends on density |
| **Top-K caching** | Store only top 5–10 queries per node instead of full subtree counts | Reduces traversal but adds per-node storage |
| **Prefix truncation** | Limit max prefix length to 50 chars | Bounds trie depth |

### Trie Serialization for Storage

**Option 1: Document store (MongoDB)**
```json
{
  "snapshot_id": "2024-W42",
  "created_at": "2024-10-14T00:00:00Z",
  "trie_data": "<serialized binary>"
}
```

**Option 2: Key-value store (hash table form)**
```
Key: prefix string     Value: top-K queries + frequencies

"be"  → [("best", 35), ("bet", 29), ("bee", 20), ("be", 15), ("beer", 10)]
"bee" → [("bee", 20), ("beer", 10), ("beef", 8), ("beep", 5), ("bees", 3)]
"bes" → [("best", 35), ("best buy", 28), ("beside", 12), ("bespoke", 8)]
```

### Data Gathering Pipeline

```
Raw Query Logs (append-only, not indexed)
    │
    ▼
Aggregation Service
    ├── Real-time apps (Twitter): aggregate per minute/hour
    └── General apps (Google): aggregate weekly
    │
    ▼
Aggregated Data Table
    │  query  │    time    │ frequency │
    │  tree   │ 2024-10-01 │   12000   │
    │  tree   │ 2024-10-08 │   15000   │
    │
    ▼
Workers (async, scheduled)
    ├── Build new trie from aggregated data
    ├── Serialize and store in Trie DB
    └── Update Trie Cache (weekly snapshot)
```

### Sharding Strategy

**Naive approach:** shard by first character
- 26 shards for a-z
- Problem: uneven distribution (`c`, `s` much more common than `x`, `z`)

**Smart approach:** shard map manager

```
Shard Map (based on historical query volume):
    Shard 1: a-b        (high volume)
    Shard 2: c           (high volume — gets its own shard)
    Shard 3: d-f
    Shard 4: g-k
    Shard 5: l-n
    Shard 6: o-r
    Shard 7: s           (high volume — gets its own shard)
    Shard 8: t-z
```

Shard assignment based on actual query distribution, not alphabetical uniformity.

### Multi-Language & Trending Support

| Challenge | Solution |
|-----------|---------|
| Multiple languages | Store Unicode characters in trie nodes |
| Per-country results | Build separate tries per country; store in regional CDN |
| Trending / real-time queries | Reduce working set by sharding; weight recent queries higher; use stream processing (Kafka, Spark Streaming, Storm) |

---

## 5. Video Transcoding Pipeline — Detail

### Codec Comparison

| Codec | Released | Compression vs H.264 | CPU Cost | Adoption |
|-------|----------|---------------------|----------|----------|
| H.264 (AVC) | 2003 | Baseline | Low | Universal — every device/browser |
| H.265 (HEVC) | 2013 | ~50% smaller at same quality | 2-3× H.264 | Growing; patent licensing issues |
| VP9 | 2013 | ~30-50% smaller | 2× H.264 | YouTube, Chrome, Android; royalty-free |
| AV1 | 2018 | ~30% smaller than HEVC | 5-10× H.264 | Netflix, YouTube adopting; royalty-free |

### Container Formats

| Container | Extension | Streaming Support | Notes |
|-----------|-----------|------------------|-------|
| MP4 | `.mp4` | Progressive download; fragmented MP4 for DASH | Most universal |
| WebM | `.webm` | Yes | VP8/VP9 + Vorbis/Opus; open format |
| MPEG-TS | `.ts` | HLS segments | Apple ecosystem standard |
| MKV | `.mkv` | Limited | Flexible; less browser support |

### Bitrate Ladder (Adaptive Streaming)

A typical bitrate ladder for a video streaming service:

| Resolution | Bitrate (video) | Total (video + audio) | Target |
|------------|----------------|----------------------|--------|
| 240p | 300 kbps | 400 kbps | Very slow connections |
| 360p | 500 kbps | 600 kbps | Mobile on 3G |
| 480p | 1,000 kbps | 1,200 kbps | Standard mobile |
| 720p | 2,500 kbps | 2,800 kbps | HD mobile / tablet |
| 1080p | 5,000 kbps | 5,500 kbps | Desktop / smart TV |
| 1440p (2K) | 8,000 kbps | 8,500 kbps | High-end displays |
| 2160p (4K) | 16,000 kbps | 17,000 kbps | 4K TVs |

Client-side player selects appropriate bitrate based on available bandwidth and buffer status.

### DAG Scheduler — Implementation Detail

```
DAG definition (from preprocessor):

  ┌──────────┐
  │ Original │
  │  Video   │
  └────┬─────┘
       │
  ┌────┴────────────────────┐
  │           │              │
  ▼           ▼              ▼
┌─────┐   ┌───────┐    ┌──────────┐
│Video│   │ Audio │    │ Metadata │     ← Stage 1: Split
└──┬──┘   └───┬───┘    └──────────┘
   │          │
   ├─────┐    │
   ▼     ▼    ▼
┌─────┐┌────┐┌─────┐
│Encode││Thum││Audio│                    ← Stage 2: Process
│240p ││b-  ││Enc  │
│360p ││nail││     │
│480p ││    ││     │
│720p ││    ││     │
│1080p││    ││     │
└─────┘└────┘└─────┘
   │     │    │
   └─────┴────┘
         │
    ┌────▼────┐
    │ Package │                          ← Stage 3: Merge
    │ (mux)   │
    └────┬────┘
         │
    ┌────▼────┐
    │ Upload  │                          ← Stage 4: Distribute
    │ to CDN  │
    └─────────┘
```

**Scheduler algorithm:**
1. Topological sort of DAG nodes
2. Group into stages (nodes with no inter-dependencies within stage)
3. Within each stage, submit all tasks to resource manager in parallel
4. Wait for stage completion before starting next stage
5. Handle failures: retry failed tasks, regenerate DAG if structure error

### Resource Manager Queues

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Task Queue    │    │  Worker Queue   │    │  Running Queue  │
│ (priority heap) │    │ (utilization)   │    │ (active jobs)   │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│ encode_720p (P1)│    │ Worker A: 30%   │    │ Task X → Wkr B  │
│ encode_480p (P2)│    │ Worker B: 80%   │    │ Task Y → Wkr C  │
│ thumbnail  (P3) │    │ Worker C: 50%   │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘

Task Scheduler:
  1. Pop highest-priority task from Task Queue
  2. Find least-utilized worker from Worker Queue
  3. Assign task to worker → move to Running Queue
  4. On completion → remove from Running Queue, free worker
```

### CDN Cost Optimization — Strategies in Detail

YouTube CDN cost estimate: 5M DAU × 5 videos × 0.3 GB × $0.02/GB = **$150,000/day**.

| Strategy | How It Works | Savings |
|----------|-------------|---------|
| **Long-tail optimization** | Only top ~20% of videos on CDN; rest on cheap blob storage | ~80% of CDN storage costs |
| **On-demand encoding** | Don't encode all resolutions for unpopular videos; encode when requested | Reduces transcoding compute ~60% |
| **Regional CDN** | Videos popular only in specific regions stay in regional CDN edge | Avoids global replication cost |
| **ISP partnerships** | Co-locate servers in ISP data centers (Netflix Open Connect model) | Reduces bandwidth charges; better latency |
| **Client-side caching** | Aggressive cache headers for video segments already watched | Reduces repeat CDN hits |
| **Compression evolution** | Migrate to AV1/VP9 from H.264 | 30-50% bandwidth reduction |

### Live Streaming vs VOD Comparison

| Aspect | Video on Demand (VOD) | Live Streaming |
|--------|----------------------|----------------|
| Latency tolerance | Seconds to minutes | Sub-second to few seconds |
| Transcoding | Offline — full pipeline | Real-time — segment-by-segment |
| Storage | Pre-encoded all resolutions | Segments stored after broadcast |
| CDN strategy | Cache full video | Cache latest segments only |
| Protocol | MPEG-DASH, HLS | Low-latency HLS, WebRTC, CMAF |
| Error recovery | Re-request segment | Skip / buffer |

### Live Streaming Flow (from ByteByteGo)

```
Streamer → Encoder → Point-of-Presence (nearest)
                          ↓
                    Transcoding (real-time, multiple resolutions)
                          ↓
                    Segment into chunks (few seconds each)
                          ↓
                    Package as HLS manifest + chunks
                          ↓
                    CDN cache (edge servers)
                          ↓
                    Viewer's video player
                          ↓
              Optional: store to S3 for replay
```

---

## Interview Cheat Sheet

### Estimation Templates

| System | Key Numbers |
|--------|------------|
| URL Shortener | 100M URLs/day; 1,160 write QPS; 11,600 read QPS; 365TB over 10 years |
| Web Crawler | 1B pages/month; 400 QPS; 500TB/month; 30PB over 5 years |
| News Feed | 10M DAU; 5000 friends per user; fan-out: push for normal, pull for celebrities |
| Autocomplete | 10M DAU; 24K QPS; 48K peak; 0.4GB new data/day; < 100ms response |
| YouTube | 5M DAU; 150TB/day upload; $150K/day CDN cost; 5 videos watched/user/day |

### Common Follow-Up Questions

| System | Likely Follow-Up |
|--------|-----------------|
| URL Shortener | Rate limiting? Database sharding? Analytics integration? |
| Web Crawler | Server-side rendering for JS-heavy sites? Anti-spam filtering? |
| News Feed | ML-based ranking? Multi-datacenter? Database scaling (SQL vs NoSQL)? |
| Autocomplete | Multi-language? Trending / real-time queries? Spell correction? |
| YouTube | Live streaming? DRM details? Recommendation system? |
