# System Design Case Studies

## The System Design Process

Every system design follows the same framework, whether you are designing a URL shortener or a global chat platform. The steps are:

1. **Requirements clarification** — What exactly are we building? What are the constraints?
2. **Back-of-the-envelope estimation** — How much storage, bandwidth, and compute do we need?
3. **High-level architecture** — What are the major components and how do they communicate?
4. **Data model** — What does the schema look like? Which database(s) do we use?
5. **API design** — What endpoints do clients call?
6. **Deep dives** — Scale bottlenecks, edge cases, failure modes

This document walks through three classic systems using this framework, then examines real-world architectures from Twitter and WhatsApp.

---

## Case Study 1: URL Shortener

### Requirements

**Functional:**
- Given a long URL, generate a short URL (e.g., `https://short.ly/abc123`)
- Given a short URL, redirect to the original long URL
- Optional: custom short codes, expiration, click analytics

**Non-functional:**
- Very high read throughput (redirects are far more common than creates)
- Low latency redirects (under 50ms)
- High availability (a broken shortener means broken links everywhere)

**Scale assumptions:**
- 100 million URLs created per month
- Read:write ratio of 100:1 (10 billion redirects per month)

### Estimation

**Storage:**
- Per URL: long URL (500 bytes avg) + short code (7 bytes) + metadata (50 bytes) = ~560 bytes
- 100M URLs/month x 12 months x 5 years = 6 billion URLs
- 6B x 560 bytes = ~3.4 TB total storage over 5 years
- Verdict: Fits on a single database with SSD. No sharding needed initially.

**Throughput:**
- Writes: 100M/month = ~40 writes/sec (very low)
- Reads: 10B/month = ~3,800 reads/sec average, ~10,000 reads/sec peak
- Verdict: A single server with caching handles this easily.

**Short code space:**
- Using Base62 (a-z, A-Z, 0-9) with 7 characters: 62^7 = 3.5 trillion possible codes
- At 100M URLs/month, this lasts thousands of years. No collision concerns.

### Architecture

```
┌─────────┐     ┌──────────┐     ┌────────────┐     ┌──────────────┐
│  Client  │────→│   CDN    │────→│ API Server │────→│   Database   │
│          │     │ (cached  │     │            │     │ (PostgreSQL) │
│          │←────│ redirects│←────│  + Redis   │←────│              │
└─────────┘     └──────────┘     └────────────┘     └──────────────┘
```

**Components:**
- **CDN:** Caches redirect responses (301/302) at the edge. For popular short URLs, the CDN serves the redirect without ever hitting the origin.
- **API Server:** Handles URL creation and redirect lookups. Stateless — any instance can handle any request.
- **Redis Cache:** Caches short-code-to-URL mappings. With 3.4 TB of total data but a power-law distribution (a small fraction of URLs get most traffic), a 50 GB Redis instance caches the hot set with 95%+ hit rate.
- **PostgreSQL:** Stores all URL mappings, metadata, and analytics data.

### Data Model

```sql
CREATE TABLE urls (
    id          BIGSERIAL PRIMARY KEY,
    short_code  VARCHAR(10) UNIQUE NOT NULL,
    long_url    TEXT NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at  TIMESTAMP,         -- NULL means never expires
    user_id     BIGINT,            -- NULL for anonymous
    click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_urls_short_code ON urls (short_code);
```

The short_code column has a unique index for fast lookups during redirects. The primary key is a BIGSERIAL for internal use and efficient JOINs.

### API Design

**Create a short URL:**
```
POST /api/v1/urls
Content-Type: application/json

{
    "long_url": "https://example.com/very/long/path?with=params",
    "custom_code": "mylink",     // optional
    "expires_in_days": 30        // optional
}

Response 201:
{
    "short_url": "https://short.ly/mylink",
    "short_code": "mylink",
    "expires_at": "2026-04-23T00:00:00Z"
}
```

**Redirect:**
```
GET /abc123

Response 301:
Location: https://example.com/very/long/path?with=params
```

Use 301 (permanent redirect) if you want browsers to cache the redirect (fewer server hits). Use 302 (temporary redirect) if you need to track every click (browsers will hit the server each time).

### Short Code Generation

Two approaches:

**Approach 1: Base62 encode an auto-increment ID.**
- URL gets ID 12345 in the database
- Base62(12345) = "dnh"
- Pad to minimum length: "0000dnh"
- Pros: No collisions, simple, sequential
- Cons: Predictable (users can guess other URLs by incrementing)

**Approach 2: Random generation with collision check.**
- Generate a random 7-character Base62 string
- Check if it exists in the database; if so, regenerate
- With 3.5 trillion possible codes and 6 billion URLs, collision probability is ~0.0002%
- Pros: Unpredictable
- Cons: Requires a uniqueness check (cheap with an index)

### Key Design Decisions

- **Read-heavy workload** means caching is the primary scaling lever. Redis + CDN handles the vast majority of traffic.
- **301 vs 302** depends on whether you need click analytics. 302 forces every click through your server.
- **Database choice** is simple: this is a key-value lookup pattern. PostgreSQL works. DynamoDB or Cassandra work if you need global distribution.

---

## Case Study 2: Real-Time Chat System

### Requirements

**Functional:**
- One-on-one messaging and group chats (up to 500 members)
- Message delivery (online and offline users)
- Message history (scrollback)
- Online/offline status (presence)
- Read receipts

**Non-functional:**
- Real-time delivery (under 200ms for online users)
- 1 million concurrent WebSocket connections
- Messages must not be lost (at-least-once delivery)
- Message ordering within a conversation

**Scale assumptions:**
- 500 million users, 100 million daily active
- 1 million concurrent connections
- 50 billion messages per day (average 500 messages per active user)

### Estimation

**Message storage:**
- Average message: 200 bytes (text) + 100 bytes (metadata) = 300 bytes
- 50B messages/day x 300 bytes = 15 TB/day
- 1 year retention: ~5.5 PB
- Verdict: Needs a distributed database optimized for writes (Cassandra, ScyllaDB)

**Connection handling:**
- 1M concurrent WebSocket connections
- Each connection uses ~10 KB of memory
- 1M x 10 KB = 10 GB RAM for connections alone
- A single server can handle ~50K-100K WebSocket connections
- Verdict: Need 10-20 WebSocket gateway servers

**Message throughput:**
- 50B messages/day = ~580K messages/sec
- Verdict: Needs a high-throughput message bus (Kafka) for fanout

### Architecture

```
┌─────────┐     ┌──────────────┐     ┌────────────────┐
│  Client  │←──→│  WebSocket   │←──→ │    Message      │
│  (App)   │    │  Gateway     │     │    Service      │
└─────────┘     └──────┬───────┘     └───────┬────────┘
                       │                      │
           ┌───────────┼──────────────────────┤
           │           │                      │
           ▼           ▼                      ▼
    ┌──────────┐ ┌──────────┐         ┌──────────────┐
    │  Redis   │ │  Kafka   │         │  Cassandra   │
    │(presence,│ │(message  │         │  (message    │
    │ sessions)│ │ fanout)  │         │   history)   │
    └──────────┘ └──────────┘         └──────────────┘
```

**Components:**

**WebSocket Gateway:** Maintains persistent WebSocket connections with clients. Each gateway instance handles 50-100K connections. Stateless in the sense that any gateway can serve any user — the mapping of user-to-gateway is stored in Redis.

**Message Service:** Receives messages from senders, persists them, and publishes them to Kafka for delivery to recipients.

**Kafka:** Distributes messages to the correct WebSocket gateway instances. When User A sends a message to User B, the message goes through Kafka to the gateway instance holding User B's connection.

**Cassandra:** Stores message history. Cassandra is ideal here because:
- Write-heavy workload (messages are written far more than read)
- Natural partitioning by conversation ID
- Time-ordered data within a partition (messages in a chat sorted by timestamp)
- Horizontal scaling by adding nodes

**Redis:** Stores session data (which gateway holds each user's connection) and presence information (online/offline status with TTL).

### Data Model (Cassandra)

```sql
-- Messages partitioned by conversation, ordered by time
CREATE TABLE messages (
    conversation_id UUID,
    message_id      TIMEUUID,
    sender_id       UUID,
    content         TEXT,
    message_type    TEXT,         -- 'text', 'image', 'file'
    created_at      TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Conversations per user (for the chat list screen)
CREATE TABLE user_conversations (
    user_id             UUID,
    last_message_at     TIMESTAMP,
    conversation_id     UUID,
    conversation_name   TEXT,
    last_message_preview TEXT,
    unread_count        INT,
    PRIMARY KEY (user_id, last_message_at)
) WITH CLUSTERING ORDER BY (last_message_at DESC);
```

The `messages` table partitions by conversation_id so all messages in a chat are co-located on the same Cassandra node. Within a partition, messages are sorted by time (descending for efficient "load recent messages" queries).

### Message Delivery Flow

**Sending a message (User A to User B):**

1. User A's client sends the message over WebSocket to a gateway
2. The gateway forwards it to the Message Service, which persists it in Cassandra
3. The Message Service publishes the message to Kafka (key: `conversation_id`)
4. A consumer looks up User B's gateway in Redis
5. If online: push over WebSocket. If offline: send a push notification (APNs/FCM)

Group messages follow the same flow for every member. Kafka handles the fanout.

### Presence System

Online/offline status uses Redis with TTL. On connect, set `presence:{user_id} = {gateway_id}` with a 30-second TTL. The client heartbeats every 20 seconds, resetting the TTL. On disconnect or crash, the key expires and the user appears offline.

### API Design

**WebSocket messages:** JSON frames for send, receive, read receipts, and typing indicators — each with a `type` field, `conversation_id`, and relevant payload.

**REST endpoints (non-real-time):**
```
GET  /api/v1/conversations                    -- List user's conversations
GET  /api/v1/conversations/{id}/messages      -- Message history (paginated)
POST /api/v1/conversations                    -- Create a group conversation
```

---

## Case Study 3: News Feed System

### Requirements

**Functional:**
- Users see a feed of posts from people they follow
- Users can post text, images, and videos
- Feed is personalized (ranked, not purely chronological)

**Non-functional:**
- Feed loads in under 500ms
- 500 million users, 200 million daily active
- Average user follows 200 accounts
- Peak: 50,000 feed requests per second

**Scale assumptions:**
- 200M DAU x average 10 feed loads per day = 2 billion feed loads per day
- 10M new posts per day
- Average post seen by 100-500 followers

### The Core Problem: Fan-Out

When User A creates a post, how do we get it into all their followers' feeds? This is the fan-out problem, and it has two fundamental approaches.

### Fan-Out on Write (Push Model)

When a user posts, immediately write that post reference into every follower's pre-computed feed cache.

- **Pro:** Feed reads are instant (pre-computed cache lookup)
- **Con:** Celebrity problem — a user with 50M followers generates 50M writes per post. Wasted work for inactive followers.

### Fan-Out on Read (Pull Model)

When a user opens their feed, compute it on the fly by fetching recent posts from all followed users and merging them.

- **Pro:** No wasted computation for inactive users. Celebrity posts written once.
- **Con:** Slow feed generation (fetch and merge from 200+ users). High database load at read time.

### Hybrid Approach (What Facebook/Twitter/Instagram Use)

The production solution combines both approaches based on the user who is posting:

- **Regular users (< 10K followers):** Fan-out on write. Pre-compute followers' feeds at post time. This covers the vast majority of users.
- **Celebrities/influencers (> 10K followers):** Fan-out on read. Do NOT write into millions of feeds. Instead, when a follower loads their feed, merge the pre-computed feed with real-time queries for celebrity posts.

```
User opens feed
    │
    ├──→ Read pre-computed feed from cache (posts from regular users)
    │
    ├──→ Fetch recent posts from followed celebrities (on-the-fly)
    │
    ▼
Merge, rank, return
```

This hybrid eliminates the celebrity problem while keeping feed reads fast for the common case (most content comes from regular users).

### Architecture

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│  Client  │────→│  API Gateway │────→│ Feed Service│
└─────────┘     └──────────────┘     └──────┬──────┘
                                            │
                    ┌───────────────────────┤
                    │           │            │
                    ▼           ▼            ▼
             ┌──────────┐ ┌─────────┐ ┌──────────┐
             │  Feed    │ │  Post   │ │  Social  │
             │  Cache   │ │  Store  │ │  Graph   │
             │ (Redis)  │ │ (DB)    │ │  Service │
             └──────────┘ └─────────┘ └──────────┘
```

**Feed Cache (Redis):** Stores pre-computed feeds as sorted sets. Each user's feed is a sorted set where the score is the post timestamp (or ranking score). `ZREVRANGE user:123:feed 0 19` returns the 20 most recent items instantly.

**Post Store:** Stores the actual post content. PostgreSQL or Cassandra depending on scale. Posts are fetched by ID after the feed service determines which post IDs to show.

**Social Graph Service:** Stores follow relationships. Used to determine a user's followers (for fan-out on write) and a user's following list (for fan-out on read of celebrity posts).

### Feed Ranking

Modern feeds are ranked, not chronological. The ranking pipeline: candidate generation (collect posts from cache + celebrity queries), scoring (ML model based on relevance, recency, relationship, content type), filtering (already seen, muted, policy violations), and final ranking with diversity rules. This pipeline runs in under 200ms for the typical case.

---

## Real-World Deep Dives

### Twitter's Timeline Architecture

Twitter's timeline went through multiple architectural generations:

**Generation 1 — Pull model (2006-2009):**
When you opened Twitter, a query fetched tweets from all accounts you follow, sorted by time. As Twitter grew, this became too slow — a user following 500 accounts triggered 500 database lookups per feed load.

**Generation 2 — Push model (2009-2012):**
Fan-out on write. When a user tweets, the tweet ID is written into every follower's home timeline cache (a Redis list). Feed reads became a simple Redis lookup. This worked until users like Lady Gaga (50M+ followers) joined. A single tweet from a celebrity triggered 50 million Redis writes, taking minutes to fan out.

**Generation 3 — Hybrid (2012-present):**
Regular users: fan-out on write (push into follower timelines). Celebrity users (configurable threshold): fan-out on read (fetch their tweets at read time and merge with the pre-computed timeline). The threshold is dynamic and based on the follower count.

**Key optimization:** Timelines store tweet IDs only (not full objects). Tweet content is fetched separately and cached. Timelines are pre-computed to a fixed depth (~800 tweets). Muted/blocked accounts are filtered at read time rather than maintained in the push pipeline.

### WhatsApp Architecture: 2 Billion Users, Minimal Servers

WhatsApp's architecture is a case study in efficiency and simplicity.

**Technology choices:** Erlang/OTP for the messaging backend (designed for telecom, excels at millions of concurrent lightweight processes — a single node handles 2-3M connections), FreeBSD tuned for high connection counts, and Mnesia (Erlang's built-in distributed database) for session data.

**Architecture principles:**

- **Store and forward:** Messages are stored temporarily on the server and forwarded to the recipient. Once delivered, they are deleted. WhatsApp does not store long-term message history — messages live on devices.
- **End-to-end encryption (Signal Protocol):** The server never sees message content, simplifying architecture to a pure routing layer for opaque encrypted blobs.
- **Connection efficiency:** Single persistent connection per user with minimal keepalive packets, delta-based presence updates, and compressed payloads.

**Scale numbers:** 2 billion users, 100 billion messages/day at peak, approximately 50 engineers and a few hundred servers before Facebook acquisition. The efficiency comes from Erlang's concurrency model, store-and-forward with deletion (minimal storage), and the server doing minimal per-message processing.

**Lesson:** Choosing the right technology (Erlang), the right architecture (store-and-forward), and deliberately limiting scope (no server-side search or long-term storage) enables extraordinary scale with minimal infrastructure.

---

## Cross-Cutting Design Considerations

These apply to all three case studies and to most system designs:

- **Idempotency:** Clients will retry failed requests. Use client-generated idempotency keys so the server deduplicates on retry (no double-sent messages, no duplicate short URLs).
- **Rate limiting:** All public APIs need rate limiting at the API gateway (token bucket is standard).
- **Monitoring:** Track latency percentiles (p50, p95, p99), error rates, queue/consumer lag, cache hit rates, and resource saturation for every service.
- **Graceful degradation:** When a component fails, degrade rather than crash. If the presence service is down, messages still deliver. If ranking is down, show a chronological feed.

## Key Takeaways

1. Start every design with requirements and estimation. The numbers determine the architecture.
2. The fan-out problem (push vs pull vs hybrid) is central to any social/feed system. The hybrid approach is the industry standard.
3. Technology choices matter enormously. WhatsApp's use of Erlang is not incidental — it is the reason they serve 2 billion users with a small team.
4. Real systems evolve through multiple architectures. Twitter's timeline went through three generations. Design for the current scale with a clear path to the next.
5. Caching is the primary scaling lever for read-heavy systems (URL shortener, news feed). Message queues and streaming are the primary scaling lever for write-heavy systems (chat).
6. Every system design involves trade-offs. Document them explicitly so future engineers understand why decisions were made.
