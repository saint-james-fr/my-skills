---
name: system-design-real-time-systems
description: Real-time communication patterns — chat systems, push notifications, WebSocket, long polling, SSE, online presence, and service discovery. Use when designing chat applications, notification systems, real-time features, or choosing between polling, WebSocket, and server-sent events.
---

# Real-Time Systems Design

## 1. Communication Protocols Comparison

| Protocol | Mechanism | Direction | Connection | Latency | Best use case | Limitations |
|----------|-----------|-----------|------------|---------|---------------|-------------|
| **HTTP polling** | Client sends requests at fixed interval | Client → Server | New connection each time | High (interval-bound) | Simple dashboards, low-frequency updates | Wasteful — most responses empty; server load scales with poll rate |
| **Long polling** | Client holds connection open until server has data or timeout | Client → Server (server holds) | Reused per cycle | Medium | Chat fallback, moderate update rates | Sender/receiver may hit different servers; no good disconnect detection; periodic reconnects after timeout |
| **WebSocket** | Full-duplex over single TCP connection (HTTP upgrade handshake) | Bidirectional | Persistent | Low | Chat, gaming, collaborative editing, real-time location | Stateful — connection management at scale; firewall/proxy issues rare (ports 80/443) |
| **SSE (Server-Sent Events)** | Server pushes over HTTP; `text/event-stream` | Server → Client | Persistent (HTTP) | Low | Live feeds, notifications, stock tickers | Unidirectional only; limited browser connections per domain (~6); no binary data |

**Decision heuristic:**
- Need bidirectional + low latency → **WebSocket**
- Server push only, text data → **SSE**
- Fallback / simple infra → **Long polling**
- Non-real-time, tolerance for delay → **HTTP polling**

```
       Polling               Long Polling            WebSocket              SSE

Client ──req──▶ Server   Client ──req──▶ Server   Client ◀══ws══▶ Server   Client ◀──stream── Server
Client ◀─resp─ Server         (server holds)        (persistent, full-      (persistent, server
Client ──req──▶ Server   Client ◀─resp─ Server      duplex after upgrade)    push only)
       ... repeat        Client ──req──▶ ...
```

## 2. Chat System Design

### Architecture split

| Layer | Technology | Responsibility |
|-------|-----------|----------------|
| **Stateless services** | HTTP behind LB | Login, signup, user profile, settings |
| **Stateful service** | WebSocket servers | Real-time messaging — client maintains persistent connection |
| **Third-party** | Push notification (APNs/FCM) | Notify offline users |

Use HTTP for sender side (proven at Facebook scale), WebSocket for receiver side. Simplification: use WebSocket for both directions.

### High-level components

```
                    ┌──────────┐
                    │   API    │  ← signup, login, profile (HTTP)
                    │ Servers  │
                    └──────────┘
┌────────┐    ws    ┌──────────┐     ┌─────────┐
│ Client ├─────────▶│  Chat    │────▶│  K-V    │  ← message store (HBase, Cassandra)
│        │◀─────────│ Servers  │     │  Store  │
└────────┘          └────┬─────┘     └─────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
        ┌──────────┐ ┌────────┐ ┌────────┐
        │ Presence │ │  ID    │ │  Push  │
        │ Servers  │ │  Gen   │ │  Notif │
        └──────────┘ └────────┘ └────────┘
```

### Storage choice

| Data type | Storage | Reason |
|-----------|---------|--------|
| Generic (user, settings, friends) | Relational DB | ACID, well-understood |
| Chat messages | Key-value (HBase, Cassandra) | Easy horizontal scaling; low-latency access; handles long tail well; 1:1 read/write ratio |

### Data models

**1-on-1 message table:**

| Column | Type |
|--------|------|
| message_id (PK) | bigint |
| from_user_id | bigint |
| to_user_id | bigint |
| content | text |
| created_at | timestamp |

**Group message table:**

| Column | Type |
|--------|------|
| channel_id (partition key) | bigint |
| message_id (sort key) | bigint |
| from_user_id | bigint |
| content | text |
| created_at | timestamp |

`message_id` must be unique and sortable by time. Options: Snowflake (global), or local sequence number (unique within a channel — simpler, sufficient).

### Message sync across devices

Each device tracks `cur_max_message_id`. New messages = those with `message_id > cur_max_message_id` and `recipient_id = current_user_id`.

### Small group vs large group chat

| Aspect | Small group (≤100–500) | Large group (1000+) |
|--------|----------------------|-------------------|
| Delivery | Copy message to each member's **message sync queue** (inbox) | Fan-out on read; don't copy per member |
| Sync | Each client reads own inbox | Client fetches from shared channel store |
| Example | WeChat (cap 500) | Discord channels |
| Trade-off | Simple sync, higher storage | Complex read logic, lower storage |

### 1-on-1 message flow

1. User A sends message to Chat server 1
2. Chat server 1 gets `message_id` from ID generator
3. Message sent to message sync queue
4. Message stored in K-V store
5. **User B online** → forwarded to Chat server 2 via WebSocket
6. **User B offline** → push notification via PN servers

## 3. Push Notification System

### Notification types

| Channel | Provider | Token/Address | Payload format |
|---------|----------|---------------|----------------|
| iOS push | APNs | Device token | JSON (`aps.alert.title/body`) |
| Android push | FCM | Registration token | JSON |
| SMS | Twilio, Nexmo | Phone number | Text |
| Email | SendGrid, Mailchimp | Email address | HTML/text |

### Contact info gathering

On app install / signup → API servers collect device tokens, phone numbers, email → store in DB.

| Table | Key columns |
|-------|-------------|
| `user` | user_id, email, phone |
| `device` | device_id, user_id, device_token, platform |

One user → many devices → push to all.

### High-level architecture (improved)

```
┌────────────┐     ┌──────────────┐     ┌──────────┐
│ Service 1  │────▶│ Notification │────▶│ Message  │
│ Service 2  │     │   Servers    │     │  Queues  │
│ Service N  │     │              │     │(per type)│
└────────────┘     └──────┬───────┘     └────┬─────┘
                          │                  │
                   ┌──────▼───────┐   ┌──────▼─────┐
                   │ Cache + DB   │   │  Workers   │
                   │(user, device,│   │(pull+send) │
                   │ templates)   │   └──────┬─────┘
                   └──────────────┘          │
                                      ┌──────▼─────┐
                                      │ 3rd Party  │
                                      │(APNs, FCM, │
                                      │ Twilio...) │
                                      └──────┬─────┘
                                      ┌──────▼─────┐
                                      │  Devices   │
                                      └────────────┘
```

Each notification type gets its own message queue → outage in one channel doesn't affect others.

### Flow

1. Service calls notification server API
2. Server fetches user info, device token, notification settings from cache/DB
3. Event enqueued to type-specific queue (iOS PN queue, SMS queue, etc.)
4. Worker pulls event from queue
5. Worker sends to third-party service
6. Third-party delivers to device

### Reliability patterns

| Pattern | Implementation |
|---------|---------------|
| **Prevent data loss** | Persist notification in DB before sending; notification log for retry |
| **Deduplication** | Check `event_id` before processing — if seen, discard |
| **Retry** | On third-party failure, re-enqueue; alert devs if persistent failure |
| **Rate limiting** | Cap notifications per user per time window to prevent opt-out |
| **Templates** | Pre-formatted notification bodies with parameter substitution |
| **Notification settings** | `user_id + channel + opt_in` table; check before sending |
| **Security** | `appKey` / `appSecret` for authenticated API access |
| **Monitoring** | Track queue depth — if growing, add workers |

### Events tracking

```
Notification created → Sent → Delivered → Opened → Clicked
```

Integrate with analytics service for open rate, click rate, engagement metrics.

## 4. Online Presence

### Heartbeat mechanism

Naive approach (mark offline on disconnect, online on reconnect) → bad UX for flaky connections (tunnels, elevators).

**Solution: heartbeat.**
- Client sends heartbeat every *x* seconds (e.g., 5s)
- Server considers user online if heartbeat received within timeout (e.g., 30s)
- After timeout with no heartbeat → mark offline

```
Client ──♥──♥──♥──╳─────────────── (30s timeout) ──▶ status = offline
         5s  5s  5s  (disconnect)
```

### Status transitions

| Event | Action |
|-------|--------|
| **Login** | WebSocket established → save `online` + `last_active_at` in KV store |
| **Logout** | Explicit → set `offline` in KV store |
| **Disconnect** | Heartbeat timeout → set `offline`; reconnect within timeout → stay `online` |

### Status fanout for friends

Publish-subscribe model: each friend pair has a channel (A-B, A-C, A-D).

When User A's status changes → publish to channels A-B, A-C, A-D.
Friends B, C, D subscribe to respective channels → receive status updates via WebSocket.

| Scale | Approach |
|-------|----------|
| Small group (≤500) | Pub/sub fanout to all friends (WeChat model) |
| Large group (1000+) | Fetch status on demand — when user enters group or refreshes |

## 5. Service Discovery

### Problem

Client needs to know which WebSocket server to connect to. Servers are stateful (persistent connections) — can't just round-robin.

### Solution: ZooKeeper

```
1. User A logs in
2. Load balancer → API server (auth)
3. Service discovery (ZooKeeper) picks best chat server
   Criteria: geographic proximity, server capacity, current load
4. Server info returned to client
5. Client opens WebSocket to assigned server
```

ZooKeeper registers all available chat servers and maintains a live registry. Coordinates with chat service to avoid overloading any single server.

### Connection management at scale

- 1M concurrent users × ~10KB memory per connection ≈ 10GB on one server (feasible but SPOF)
- Production: distribute across many WebSocket servers
- When a chat server goes down → ZooKeeper provides new server → clients reconnect
- Connection draining: mark node as "draining" at LB → no new connections → wait for existing to close → remove node

## 6. Nearby Friends (Real-Time Location)

### Architecture

```
┌────────┐    ws    ┌────────────┐    pub    ┌──────────────┐
│ Client ├─────────▶│ WebSocket  ├──────────▶│ Redis Pub/Sub│
│        │◀─────────│  Servers   │◀──────────│   Cluster    │
└────────┘          └─────┬──────┘    sub    └──────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐ ┌─────────┐ ┌──────────┐
        │ Redis    │ │Location │ │  User    │
        │ Location │ │ History │ │    DB    │
        │  Cache   │ │   DB    │ │          │
        └──────────┘ └─────────┘ └──────────┘
```

### Location update flow

1. Client sends location update via WebSocket
2. WebSocket server saves to location history DB
3. Updates Redis location cache (refreshes TTL)
4. Publishes to user's channel on Redis pub/sub
5. Redis broadcasts to all subscribers (user's friends' connection handlers)
6. Each subscriber computes distance; if within radius → forward to friend's client
7. If outside radius → drop update

### Redis pub/sub for fan-out

Each user gets a dedicated channel. On initialization, client subscribes to all friends' channels (active + inactive — cheap, no CPU used for idle channels).

| Metric | Calculation |
|--------|-------------|
| Channels | 100M users × 10% active = 10M channels |
| Memory per channel | ~20 bytes × 100 friends = 2KB |
| Total memory | ~200GB → 2 Redis servers |
| Location updates/sec | 10M users / 30s interval = 334K |
| Fan-out | 334K × 400 friends × 10% online = 13M pushes/sec |
| CPU bottleneck | ~100K pushes/server → need ~130 Redis servers |

**Bottleneck is CPU, not memory.** Shard channels by user_id across distributed Redis cluster.

### Geohash-based channels (nearby random people)

For non-friend proximity: create pub/sub channels per geohash grid. Users within same grid subscribe to same channel. Subscribe to user's grid + 8 surrounding grids to handle border cases.

### Scaling WebSocket servers

- Stateful → careful scaling: mark as "draining" before removal
- Auto-scale based on usage, but drain existing connections first
- Over-provision to handle daily peaks without frequent resizing

### Service discovery for pub/sub cluster

Store hash ring of Redis pub/sub servers in ZooKeeper/etcd:

```
Key: /config/pub_sub_ring
Value: ["p_1", "p_2", "p_3", "p_4"]
```

WebSocket servers cache hash ring locally, subscribe to updates. Consistent hashing determines which pub/sub server handles each channel.

**Resizing:** update hash ring → mass resubscription spike → do during lowest traffic. Treat pub/sub cluster as **stateful** — avoid frequent resizing.

## 7. Reliability Patterns

### Message ordering

| Approach | Scope | Trade-off |
|----------|-------|-----------|
| Snowflake ID | Global | Complex infrastructure; globally unique + time-sortable |
| Local sequence number | Per-channel | Simpler; unique only within channel — sufficient for chat |
| `created_at` timestamp | — | **Not reliable** — two messages can share same timestamp |

### Delivery guarantees

| Guarantee | How |
|-----------|-----|
| **At-least-once** (notifications) | Persist before send; retry on failure; accept duplicates |
| **Deduplication** | `event_id` check — discard if already processed |
| **Exactly-once** | Not achievable in distributed systems; approximate with dedup |

### Retry mechanisms

```
Send attempt → failure → re-enqueue to message queue
                         ↓
                    retry (up to N times)
                         ↓
                    if still failing → alert on-call
```

### Monitoring

| Metric | Action |
|--------|--------|
| Queue depth growing | Add workers |
| Delivery latency increasing | Check third-party service health |
| High error rate | Trigger circuit breaker; alert team |
| Connection count per WebSocket server | Auto-scale or drain overloaded nodes |

## Quick Reference: System Design Checklist

| Component | Key decision |
|-----------|-------------|
| Protocol | WebSocket for bidirectional; SSE for server-push only; long polling as fallback |
| Chat storage | Key-value store (HBase/Cassandra) — not relational for message data |
| Group chat | Small (≤500): copy to each inbox. Large: fan-out on read |
| Notifications | Separate queues per channel type; persist before send; dedup by event_id |
| Presence | Heartbeat with timeout; pub/sub fanout for small groups; on-demand for large |
| Service discovery | ZooKeeper/etcd for WebSocket server assignment; consistent hashing for pub/sub |
| Location updates | Redis pub/sub per-user channels; geohash channels for strangers |
| Scaling WebSocket | Stateful — drain before remove; over-provision; hash ring for pub/sub routing |
