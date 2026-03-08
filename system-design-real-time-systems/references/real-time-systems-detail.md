# Real-Time Systems — Detailed Reference

## Chat System Message Flow Walkthrough

### 1-on-1 flow (User A → User B)

```
User A                Chat Server 1       ID Gen    Sync Queue    K-V Store    Chat Server 2       User B
  │                        │                │           │             │              │                │
  │── send message ───────▶│                │           │             │              │                │
  │                        │── get ID ─────▶│           │             │              │                │
  │                        │◀── msg_id ─────│           │             │              │                │
  │                        │── enqueue ────────────────▶│             │              │                │
  │                        │                │           │── store ───▶│              │                │
  │                        │                │           │             │              │                │
  │                        │     [if B online]          │             │              │                │
  │                        │── forward ────────────────────────────────────────────▶│                │
  │                        │                │           │             │              │── deliver ────▶│
  │                        │                │           │             │              │                │
  │                        │     [if B offline]         │             │              │                │
  │                        │── push notif ──▶ PN Server │             │              │                │
```

### Multi-device sync

```
User A (phone)              Chat Server 1              User A (laptop)
     │                           │                          │
     │── send message ──────────▶│                          │
     │                           │                          │
     │  cur_max_msg_id = 678     │    cur_max_msg_id = 675  │
     │                           │                          │
     │                     New message stored (msg_id = 679)│
     │                           │                          │
     │  phone pulls: ID > 678   ─┤─  laptop pulls: ID > 675│
     │  gets: [679]              │   gets: [676,677,678,679]│
     │                           │                          │
     │  cur_max_msg_id = 679     │    cur_max_msg_id = 679  │
```

Each device independently tracks its `cur_max_message_id`. On sync, device fetches all messages where:
- `recipient_id = current_user_id`
- `message_id > cur_max_message_id`

### Small group chat flow (3 members: A, B, C)

```
User A                Chat Server          Message Sync Queues
  │                       │
  │── send to group ─────▶│
  │                       │── copy to B's inbox ──▶ [Queue B]
  │                       │── copy to C's inbox ──▶ [Queue C]
  │                       │
  │                       │   B reads own inbox ◀── [Queue B]
  │                       │   C reads own inbox ◀── [Queue C]
```

Each client only checks its own inbox. Trade-off: works well up to ~500 members (WeChat limit), too expensive for larger groups.

### Large group alternative

For groups with thousands of members, switch to fan-out-on-read:
- Store message once in channel store
- Clients fetch from shared store on demand
- Avoids N copies per message

## Notification System Events Tracking

### Full event lifecycle

```
┌──────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐
│ Created  │───▶│  Queued  │───▶│   Sent    │───▶│Delivered │───▶│  Opened  │
└──────────┘    └──────────┘    └───────────┘    └──────────┘    └─────┬────┘
                                      │                               │
                                      ▼                               ▼
                                ┌───────────┐                   ┌──────────┐
                                │  Failed   │                   │ Clicked  │
                                └─────┬─────┘                   └──────────┘
                                      │
                                      ▼
                                ┌───────────┐
                                │  Retried  │
                                └───────────┘
```

### Event tracking table

| Event | Timestamp | Data captured |
|-------|-----------|---------------|
| `notification.created` | When service calls API | service_id, user_id, channel, template_id |
| `notification.queued` | When placed in MQ | queue_name, position |
| `notification.sent` | When worker sends to 3rd party | worker_id, 3rd_party_response |
| `notification.delivered` | When 3rd party confirms delivery | delivery_id, device_id |
| `notification.opened` | When user opens/views | open_timestamp, device_type |
| `notification.clicked` | When user taps CTA | click_url, conversion_id |
| `notification.failed` | When send fails | error_code, error_message, retry_count |

### Metrics derived

| Metric | Formula | Healthy range |
|--------|---------|---------------|
| Delivery rate | delivered / sent | > 95% |
| Open rate | opened / delivered | 5–15% (push), 20–30% (email) |
| Click-through rate | clicked / opened | 2–5% |
| Failure rate | failed / sent | < 1% |
| Avg queue time | avg(sent_ts - queued_ts) | < 30s |

## Notification Decision Flowchart

Based on Slack's notification model — deciding whether to actually deliver a notification.

```
Incoming notification event
        │
        ▼
┌───────────────────┐     No
│ User opted-in for │──────────▶ DROP
│ this channel?     │
└───────┬───────────┘
        │ Yes
        ▼
┌───────────────────┐     Yes
│ Rate limit        │──────────▶ DROP (or DEFER)
│ exceeded?         │
└───────┬───────────┘
        │ No
        ▼
┌───────────────────┐     Yes
│ Duplicate event   │──────────▶ DROP
│ (event_id seen)?  │
└───────┬───────────┘
        │ No
        ▼
┌───────────────────┐     Yes
│ User currently    │──────────▶ IN-APP only (suppress push)
│ active in app?    │
└───────┬───────────┘
        │ No
        ▼
┌───────────────────┐     Yes
│ DND / quiet hours │──────────▶ DEFER (schedule for later)
│ active?           │
└───────┬───────────┘
        │ No
        ▼
┌───────────────────┐
│ Send via preferred│
│ channel(s)        │
└───────────────────┘
```

### Notification settings table

```
notification_settings
├── user_id          bigint (PK)
├── channel          varchar  (push | email | sms)
├── opt_in           boolean
├── quiet_start      time     (e.g., 22:00)
├── quiet_end        time     (e.g., 08:00)
├── rate_limit       int      (max per hour)
└── updated_at       timestamp
```

## Scaling WebSocket to Millions of Connections

### Memory budget

| Component | Per connection | 1M connections | 10M connections |
|-----------|---------------|----------------|-----------------|
| TCP socket buffer | ~4 KB | 4 GB | 40 GB |
| Application state | ~6 KB | 6 GB | 60 GB |
| **Total** | **~10 KB** | **10 GB** | **100 GB** |

A single modern server can hold ~1M connections (10 GB). For 10M, need ~10+ servers.

### Connection lifecycle

```
Client                    Load Balancer              WebSocket Server
  │                            │                           │
  │── HTTP upgrade request ───▶│                           │
  │                            │── route to server ───────▶│
  │                            │   (sticky by user_id      │
  │                            │    or service discovery)   │
  │◀════════════ WebSocket established ════════════════════│
  │                            │                           │
  │──── heartbeat ────────────────────────────────────────▶│
  │──── heartbeat ────────────────────────────────────────▶│
  │                            │                           │
  │    (client moves to tunnel / loses signal)             │
  │                            │                           │
  │                            │       heartbeat timeout ──│
  │                            │       mark user offline   │
  │                            │                           │
  │── reconnect ──────────────▶│                           │
  │                            │── may route to new server─▶│
  │◀════════════ new WebSocket ════════════════════════════│
```

### Server draining for deployments

```
1. Mark server as "draining" in load balancer
2. LB stops routing new connections to this server
3. Existing connections continue working
4. Wait for connections to close naturally (or timeout after N minutes)
5. Remove server from pool
6. Deploy new version on fresh server
7. Add new server to LB pool
```

### Chat server failure recovery

```
Chat Server 2 goes DOWN
        │
        ▼
ZooKeeper detects failure (heartbeat missed)
        │
        ▼
ZooKeeper updates server registry
        │
        ▼
Clients of Server 2 detect connection lost
        │
        ▼
Clients request new server from service discovery
        │
        ▼
ZooKeeper assigns Chat Server 5 (best available)
        │
        ▼
Clients establish new WebSocket to Server 5
        │
        ▼
Clients sync messages using cur_max_message_id
```

## Real-Time Location Update Architecture

### Periodic update flow (Nearby Friends)

```
Step 1:  Client ──location update──▶ Load Balancer ──▶ WebSocket Server

Step 2:  WebSocket Server saves to location history DB (async)

Step 3:  WebSocket Server updates Redis location cache
         Key: user_id → {lat, lng, timestamp}
         TTL refreshed on each update

Step 4:  WebSocket Server publishes to Redis pub/sub
         Channel: user_<id>

Step 5:  Redis pub/sub broadcasts to subscribers
         (subscribers = friends' WebSocket connection handlers)

Step 6:  Each subscriber's handler:
         - Gets update message (sender's location)
         - Reads subscriber's location (stored in handler variable)
         - Computes distance

Step 7:  If distance ≤ search radius → forward to subscriber's client
         If distance > search radius → drop
```

### Client initialization sequence

```
1. Client opens WebSocket connection
2. Sends initial location
3. Server updates location cache
4. Server stores location in connection handler variable
5. Server loads user's friend list from user DB
6. Server batch-fetches all friends' locations from Redis cache
   (missing = friend is inactive, TTL expired)
7. For each friend with location:
   - Compute distance
   - If within radius → return friend profile + location + timestamp
8. Subscribe to ALL friends' channels on Redis pub/sub
   (even inactive — channels are cheap when idle)
9. Publish user's location to own channel
   (so friends who are already online get notified)
```

### Redis pub/sub cluster management

**Hash ring stored in service discovery (ZooKeeper/etcd):**

```
/config/pub_sub_ring = ["p_1", "p_2", "p_3", "p_4"]
```

**Channel-to-server mapping:**

```
channel_user_123 ──hash──▶ p_2
channel_user_456 ──hash──▶ p_1
channel_user_789 ──hash──▶ p_4
```

**When a pub/sub server dies (e.g., p_1):**

```
1. Monitoring detects p_1 is down
2. On-call updates hash ring:
   Old: ["p_1", "p_2", "p_3", "p_4"]
   New: ["p_1_new", "p_2", "p_3", "p_4"]
3. Service discovery notifies all WebSocket servers
4. Each WebSocket handler checks its subscribed channels against new ring
5. Channels that were on p_1 → unsubscribe from p_1, resubscribe on p_1_new
6. Only channels on the replaced server are affected (minimal disruption)
```

**When scaling up (adding servers):**

```
Old: ["p_1", "p_2", "p_3", "p_4"]
New: ["p_1", "p_2", "p_3", "p_4", "p_5", "p_6"]

Impact: many channels rebalance across ring → mass resubscription
Mitigation: do during lowest traffic; over-provision to avoid frequent resizing
```

### Erlang as alternative to Redis pub/sub

Erlang/BEAM VM provides lightweight processes (~300 bytes each). Model each user as an Erlang process:

| Aspect | Redis pub/sub | Erlang |
|--------|--------------|--------|
| Channel model | Explicit channels per user | Each user = Erlang process |
| Subscription | Redis subscribe commands | Native OTP pub/sub |
| Distribution | External consistent hashing + service discovery | Built-in distribution in BEAM |
| Memory per user | ~20 bytes × subscribers | ~300 bytes per process |
| Operational | Separate Redis cluster to manage | Single Erlang application |
| Hiring | Common skills | Niche — hard to hire |

Erlang approach eliminates the Redis pub/sub cluster entirely. The Erlang processes form a mesh that routes location updates directly.

### Geohash channels for nearby strangers

```
Geohash Grid         Redis Pub/Sub Channels
┌────┬────┐
│9q8z│9q8z│          channel_9q8znd ← User 1, User 2, User 3
│ nc │ nd │          channel_9q8znc ← User 4
├────┼────┤          channel_9q8znf ← User 5
│9q8z│9q8z│          channel_9q8zne ← (empty)
│ ne │ nf │
└────┴────┘

User subscribes to own grid + 8 surrounding grids (handles border proximity).

Location update:
1. User computes geohash from lat/lng
2. Publishes to geohash channel
3. All subscribers in same grid receive update
4. Each subscriber computes distance, forwards if within radius
```
