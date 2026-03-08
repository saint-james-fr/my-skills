---
name: system-design-content-platforms
description: Content platform design — URL shortener, web crawler, news feed, search autocomplete, and video streaming (YouTube). Use when designing URL shortening services, building web crawlers, implementing news feeds with fan-out strategies, creating search autocomplete with tries, or architecting video transcoding pipelines.
---

# Content Platforms — System Design Reference

Based on *System Design Interview* (Vol. 1) by Alex Xu, Chapters 8, 9, 11, 13, 14; *ByteByteGo* archive.

---

## 1. URL Shortener

### API Design

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `api/v1/data/shorten` | POST | `{longUrl}` → returns `shortURL` |
| `api/v1/{shortUrl}` | GET | Redirect to original `longURL` |

### 301 vs 302 Redirect

| Aspect | 301 (Permanent) | 302 (Temporary) |
|--------|-----------------|-----------------|
| Browser behavior | Caches redirect — subsequent requests skip shortener | Every request hits shortener first |
| Server load | Lower — only first request reaches service | Higher — all requests reach service |
| Analytics | Harder — can't track clicks after caching | Easier — every click is trackable |
| Best for | Reducing server load | Click analytics, A/B testing |

### Hash Value Length

Characters `[0-9, a-z, A-Z]` = 62 possible. Need `62^n ≥ 365 billion` (10 years of URLs at 100M/day).

| n | Max URLs | Sufficient? |
|---|----------|-------------|
| 6 | 56.8 billion | No |
| 7 | 3.5 trillion | Yes |

**Use n = 7** for the hash value length.

### Hash Function Approaches

| | Hash + Collision Resolution | Base-62 Conversion |
|---|---|---|
| **How** | Hash URL (CRC32/MD5/SHA-1) → take first 7 chars → resolve collisions | Use unique ID generator → convert ID to base-62 |
| **Length** | Fixed (7 chars) | Variable (grows with ID) |
| **ID generator** | Not needed | Required (distributed ID gen) |
| **Collisions** | Possible — must check DB or Bloom filter | Impossible — IDs are unique |
| **Security** | Next URL unpredictable | Next URL predictable (sequential IDs) |

### Hash Algorithms Comparison

| Algorithm | Output (hex) | Length | Notes |
|-----------|-------------|-------|-------|
| CRC32 | `5cb54054` | 8 chars | Still too long, need to truncate |
| MD5 | `5a62509a84df9ee0...` | 32 chars | Truncate to 7 + collision check |
| SHA-1 | `0eeae7916c068539...` | 40 chars | Truncate to 7 + collision check |

### Deep Dive Flow (Base-62)

```
1. longURL input
2. Check DB — already shortened?
   ├─ Yes → return existing shortURL
   └─ No  → 3. Generate unique ID (distributed ID generator)
            4. Convert ID to base-62 → shortURL
            5. Save (id, shortURL, longURL) to DB
```

### URL Redirect Flow (Read-Heavy)

```
Client → LB → Web Server → Cache hit? ─── Yes → return longURL
                              │
                              No → DB lookup → populate cache → return longURL
```

Data model: `(id, shortURL, longURL)` in relational DB. Cache `<shortURL, longURL>` pairs for read-heavy workload (10:1 read/write ratio).

---

## 2. Web Crawler

### Components

```
Seed URLs → URL Frontier → HTML Downloader → DNS Resolver
                                    ↓
                            Content Parser → Content Seen? (hash dedup)
                                    ↓
                            URL Extractor → URL Filter → URL Seen? (Bloom filter)
                                    ↓
                              URL Frontier (loop)
```

| Component | Role |
|-----------|------|
| **Seed URLs** | Starting points — divide by locality or topic |
| **URL Frontier** | FIFO queue of URLs to download; manages priority + politeness |
| **HTML Downloader** | Fetches pages via HTTP; checks robots.txt first |
| **DNS Resolver** | URL → IP translation (bottleneck — cache aggressively) |
| **Content Parser** | Validates HTML; separate component to avoid slowing crawler |
| **Content Seen?** | Hash-based dedup — ~29% of web is duplicate content |
| **URL Extractor** | Parses links from HTML; converts relative → absolute URLs |
| **URL Filter** | Excludes blacklisted domains, file types, error links |
| **URL Seen?** | Bloom filter / hash table — prevents re-visiting URLs |

### BFS vs DFS

| | BFS | DFS |
|---|---|---|
| **Implementation** | FIFO queue | Stack / recursion |
| **Preferred?** | Yes | No |
| **Why** | Controllable breadth; natural for web structure | Can go infinitely deep; spider trap risk |

BFS has two problems solved by URL Frontier: (1) same-host flooding ("impolite"), (2) no priority awareness.

### URL Frontier Design

Two modules: **front queues** (priority) + **back queues** (politeness).

**Politeness (back queues):**

| Component | Function |
|-----------|----------|
| Queue router | Routes URLs so each queue has only same-host URLs |
| Mapping table | `hostname → queue` mapping |
| Per-host FIFO queues | `b1..bn` — one queue per host |
| Queue selector | Maps each worker thread to one queue |
| Worker threads | Download one page at a time per host with delay |

**Priority (front queues):**

| Component | Function |
|-----------|----------|
| Prioritizer | Computes URL priority (PageRank, traffic, update frequency) |
| Priority queues | `f1..fn` — higher-priority queues polled more often |
| Queue selector | Biased random selection toward high-priority queues |

**Freshness:** Recrawl based on update history; prioritize important pages for more frequent recrawling.

**Storage:** Hybrid — majority on disk, buffers in memory for enqueue/dequeue. Periodic flush to disk.

### Robots.txt

Always check `robots.txt` before crawling a site. Cache results — download periodically.

### Performance Optimizations

| Optimization | How |
|-------------|-----|
| Distributed crawl | Partition URL space across servers; each runs multiple threads |
| DNS caching | Maintain local DNS cache; update via cron (DNS is 10–200ms per call) |
| Locality | Place crawl servers geographically close to target hosts |
| Short timeout | Set max wait time — skip unresponsive servers |

### Robustness

| Technique | Purpose |
|-----------|---------|
| Consistent hashing | Distribute load among downloaders; easy add/remove servers |
| Checkpointing | Save crawl state to storage; restart from saved state on failure |
| Graceful shutdown | Handle exceptions without crashing the system |
| Data validation | Prevent system errors from malformed input |

### Content Deduplication

Use **simhash** or other fingerprinting to compare web page content. Character-by-character comparison is too slow at scale. Compare hash values instead.

### Spider Trap Avoidance

Set max URL length. Monitor for sites with unusually large page counts. Manual verification + custom URL filters for known traps. No universal automated solution exists.

---

## 3. News Feed System

### Two Parts

| Flow | Purpose |
|------|---------|
| **Feed publishing** | User posts → data written to cache/DB → propagated to friends' feeds |
| **Feed retrieval** | User loads feed → aggregated posts in reverse chronological order |

### APIs

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/me/feed` | POST | Publish a post (`content`, `auth_token`) |
| `/v1/me/feed` | GET | Retrieve news feed (`auth_token`) |

### Fan-out Strategies

| | Fan-out on Write (Push) | Fan-out on Read (Pull) |
|---|---|---|
| **When** | At write time — pre-compute feed | At read time — on-demand |
| **Speed** | Fast read (pre-computed) | Slow read (computed on request) |
| **Write cost** | High — push to all followers | Low — just write post |
| **Hotkey problem** | Yes — celebrities with millions of followers | No |
| **Inactive users** | Waste — compute feeds no one reads | Efficient — only compute when requested |
| **Real-time** | Yes — immediate delivery | No — delay on read |

### Hybrid Approach (Recommended)

| User type | Strategy | Reason |
|-----------|----------|--------|
| Normal users (majority) | Fan-out on write | Fast reads; manageable follower count |
| Celebrities / high-follower | Fan-out on read | Avoid hotkey problem; pull on demand |

Use consistent hashing to mitigate hotkey distribution.

### Feed Publishing Flow

```
User → LB → Web Server (auth + rate limit) → Post Service (DB + cache)
                                             → Fanout Service
                                             → Notification Service
```

**Fanout service steps:**
1. Fetch friend IDs from graph DB
2. Get friend info from user cache; apply filters (muted users, privacy settings)
3. Push friend list + post ID to message queue
4. Fanout workers write `<post_id, user_id>` to news feed cache
5. Only store IDs in cache (not full objects) — set configurable limit

### Feed Retrieval Flow

```
User → LB → Web Server → News Feed Service → News Feed Cache (post IDs)
                                             → User Cache + Post Cache (hydrate)
                                             → Return fully hydrated JSON
```

Media content (images, videos) served from CDN.

### Cache Architecture (5 Layers)

| Cache Layer | What It Stores |
|-------------|---------------|
| **News Feed** | IDs of news feed posts |
| **Content** | Post data; popular content in hot cache |
| **Social Graph** | User relationship data (friends, followers) |
| **Action** | Likes, replies, shares per user-post |
| **Counters** | Like count, reply count, follower/following counts |

---

## 4. Search Autocomplete

### Requirements

- Return top 5 suggestions within **100ms** (stuttering threshold)
- Sorted by popularity (historical query frequency)
- ~24,000 QPS (10M DAU × 10 queries × 20 chars / 86,400s); peak ~48,000

### Trie Data Structure

A tree where each node stores a character; root = empty string; each node has up to 26 children.

**Naive algorithm:**
1. Find prefix node — `O(p)`
2. Traverse subtree for valid children — `O(c)`
3. Sort children, return top-k — `O(c log c)`
Total: `O(p + c + c log c)` — too slow at scale.

**Two optimizations:**

| Optimization | Effect |
|-------------|--------|
| Limit prefix length (≤50 chars) | "Find prefix" → `O(1)` |
| Cache top-K at each node | "Return top-K" → `O(1)` |

After both optimizations: **O(1)** per query.

Trade-off: more space per node (store top-K list), but constant-time lookups.

### System Architecture

```
┌─────────────┐      ┌──────────────┐
│ Analytics   │ ──→  │ Aggregators  │ ──→ Workers ──→ Trie DB
│ Logs        │      │ (weekly)     │                    │
└─────────────┘      └──────────────┘              Trie Cache
                                                       │
Client → LB → API Servers ──→ Trie Cache ──→ response
```

| Component | Role |
|-----------|------|
| **Analytics logs** | Raw search query data (append-only) |
| **Aggregators** | Aggregate query frequencies (weekly or real-time depending on use case) |
| **Workers** | Build trie data structure asynchronously |
| **Trie DB** | Persistent storage — document store (MongoDB) or KV store (hash table form) |
| **Trie Cache** | Distributed cache; weekly snapshot from DB |
| **API servers** | Serve autocomplete requests from Trie Cache |

### Query Service Optimizations

| Optimization | How |
|-------------|-----|
| AJAX requests | No full page refresh on each keystroke |
| Browser caching | Cache suggestions client-side (`Cache-Control: private, max-age=3600`) |
| Data sampling | Log only 1 in N requests to reduce processing/storage |

### Trie Operations

| Operation | Approach |
|-----------|---------|
| **Create** | Built by workers from aggregated analytics data |
| **Update** | Option 1: Replace entire trie weekly. Option 2: Update individual nodes (slow — must propagate to ancestors) |
| **Delete** | Filter layer in front of Trie Cache removes hateful/explicit/dangerous suggestions; async cleanup in DB |

### Scaling the Trie

Shard by first character(s): `a-m` on server 1, `n-z` on server 2.

Problem: uneven distribution (more words start with `c` than `x`).

Solution: **shard map manager** — analyze historical data distribution, assign shards based on actual query volume rather than alphabetical split.

---

## 5. Video Streaming (YouTube)

### High-Level Components

| Component | Role |
|-----------|------|
| **Client** | Computer, mobile, smart TV |
| **CDN** | Stream videos from edge servers closest to user |
| **API servers** | Everything except video streaming: feed, upload URL, metadata, auth |

### Video Upload Flow

```
Client uploads video ──→ Original Storage (blob)
                              ↓
                     Transcoding Servers
                         ↓           ↓
              Transcoded Storage   Completion Queue
                    ↓                    ↓
                  CDN            Completion Handler → Metadata DB + Cache
```

Two parallel processes:
- **Flow A:** Upload video → transcode → CDN
- **Flow B:** Upload metadata (filename, size, format) → API servers → Metadata DB + cache

### Video Transcoding

**Why transcode:**
- Raw video is huge (~hundreds of GB/hour at 60fps HD)
- Different devices/browsers need different formats
- Adaptive bitrate for varying network conditions

**Two parts of encoding format:**

| Part | What | Examples |
|------|------|---------|
| Container | Basket holding video + audio + metadata | `.avi`, `.mov`, `.mp4` |
| Codec | Compression/decompression algorithm | H.264, VP9, HEVC |

### DAG Model for Transcoding Pipeline

```
Original Video
     ├── Video → Inspection → Video Encoding (multiple resolutions) → Thumbnails
     ├── Audio → Audio Encoding
     └── Metadata
```

Tasks (inspection, encoding, thumbnail, watermark) defined as DAG nodes — executed sequentially or in parallel.

### Transcoding Architecture

| Component | Role |
|-----------|------|
| **Preprocessor** | Split video into GOPs (Group of Pictures); generate DAG; cache segments |
| **DAG Scheduler** | Split DAG into stages of tasks; enqueue to resource manager |
| **Resource Manager** | Task queue + worker queue + running queue + task scheduler |
| **Task Workers** | Execute encoding/thumbnail/watermark tasks |
| **Temporary Storage** | Cache metadata in memory; blob storage for video/audio data |
| **Encoded Video** | Final output (e.g., `funny_720p.mp4`) |

**Resource manager flow:**
1. Task scheduler picks highest-priority task from task queue
2. Picks optimal worker from worker queue
3. Binds task/worker → running queue
4. Removes from running queue when done

### Streaming Protocols

| Protocol | Provider |
|----------|---------|
| **MPEG-DASH** | Moving Picture Experts Group — Dynamic Adaptive Streaming over HTTP |
| **Apple HLS** | HTTP Live Streaming — most common for live streaming |
| **Microsoft Smooth Streaming** | Microsoft |
| **Adobe HDS** | HTTP Dynamic Streaming |

### Speed Optimizations

| Optimization | How |
|-------------|-----|
| Parallel upload | Split video by GOP on client → upload chunks independently → resumable |
| Upload centers | Multiple upload centers worldwide (CDN as upload points) |
| Parallelism everywhere | Message queues between processing stages — encoding doesn't wait for download |

### Safety Mechanisms

| Mechanism | Purpose |
|-----------|---------|
| Pre-signed upload URL | Only authorized users upload to correct location (S3 pre-signed / Azure SAS) |
| DRM | Apple FairPlay, Google Widevine, Microsoft PlayReady |
| AES encryption | Encrypt video; decrypt on authorized playback |
| Visual watermarking | Overlay company logo / identifying info |

### Cost Optimization (Long-Tail Distribution)

Most videos get few views; a small percentage drives most traffic.

| Strategy | Savings |
|----------|---------|
| Serve only popular videos from CDN | Avoid CDN cost for rarely-viewed content |
| Less popular → high-capacity storage servers | Cheaper than CDN |
| Short/unpopular videos — encode on demand | Don't pre-encode all resolutions |
| Regional popularity → regional CDN only | Don't distribute globally if views are local |
| Build own CDN + ISP partnerships | Netflix model — reduce bandwidth charges |

### Error Handling

| Component | Recovery |
|-----------|----------|
| Upload error | Retry |
| Split video error | Server-side splitting fallback for old clients |
| Transcoding error | Retry |
| Preprocessor error | Regenerate DAG |
| DAG scheduler error | Reschedule task |
| Resource manager queue down | Use replica |
| Task worker down | Retry on new worker |
| API server down | Stateless — route to different server |
| Metadata cache down | Replicated — use another node |
| Metadata DB master down | Promote slave |
| Metadata DB slave down | Use another slave; bring up replacement |

---

## Quick Reference: When to Use What

| Problem | Key Pattern | Core Data Structure |
|---------|------------|-------------------|
| URL shortener | Base-62 + distributed ID generator | Hash table / relational DB + cache |
| Web crawler | BFS + URL Frontier (priority + politeness) | Bloom filter + FIFO queues |
| News feed | Hybrid fan-out (push for normal, pull for celebrities) | Graph DB + multi-layer cache |
| Search autocomplete | Trie with cached top-K per node | Trie + distributed cache |
| Video streaming | DAG transcoding pipeline + CDN | Blob storage + message queues |
