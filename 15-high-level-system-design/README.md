# 15 - High-Level System Design

## Concepts

### What is High-Level System Design?

High-level system design (HLD) is about designing the overall structure of a distributed system: what components exist, how they communicate, where data lives, and how the system scales. While low-level design focuses on the internals of a single component, HLD focuses on the big picture.

HLD answers: "If we need to handle 10 million users, what does the architecture look like?"

### Scaling Fundamentals

#### Vertical Scaling (Scale Up)
Add more resources to a single machine: more CPU, RAM, faster SSDs.

- **Pros:** Simple, no code changes, no distributed system complexity
- **Cons:** Physical limits (you can't add infinite RAM), single point of failure, expensive at the top end
- **Limit:** The largest cloud instances have ~400 vCPUs and ~24 TB RAM. Beyond that, you must scale horizontally.

#### Horizontal Scaling (Scale Out)
Add more machines and distribute the workload.

- **Pros:** Near-infinite scaling, fault tolerance (one machine dying doesn't kill the system)
- **Cons:** Distributed system complexity (consistency, network partitions, coordination)

**The practical approach:** Scale vertically until you can't (it's simpler), then scale horizontally where needed. Most applications can serve thousands of users on a single well-optimized server.

### Load Balancing

A load balancer distributes incoming requests across multiple backend servers.

```
                     ┌──→ Server 1
Client ──→ [Load    ]├──→ Server 2
            Balancer ├──→ Server 3
                     └──→ Server 4
```

**Layer 4 (Transport) vs Layer 7 (Application):**

| Feature | L4 Load Balancer | L7 Load Balancer |
|---------|-----------------|-----------------|
| **Operates on** | TCP/UDP (IP + port) | HTTP (URLs, headers, cookies) |
| **Routing** | By IP/port only | By URL path, header, cookie, content |
| **Speed** | Faster (less inspection) | Slower (parses HTTP) |
| **Features** | Simple distribution | URL-based routing, SSL termination, header manipulation |
| **Use case** | Database connections, non-HTTP | Web APIs, microservice routing |

**Load balancing algorithms:**

| Algorithm | How it works | Best for |
|-----------|-------------|----------|
| **Round Robin** | Each request goes to the next server in order | Equal-capacity servers, stateless requests |
| **Weighted Round Robin** | More requests go to higher-capacity servers | Mixed server capacities |
| **Least Connections** | Request goes to server with fewest active connections | Variable request duration |
| **IP Hash** | Same client IP always goes to the same server | Session affinity without sticky sessions |
| **Random** | Pick a random server | Simple, surprisingly effective at scale |

### Caching Strategies

Caching stores frequently accessed data closer to the consumer, reducing database load and improving response times.

**Cache levels:**

```
Client → [Browser Cache] → [CDN] → [API Gateway Cache] → [Application Cache] → [Database Cache] → [Database]
         Fastest                                                                                      Slowest
```

**Caching patterns:**

#### Cache-Aside (Lazy Loading)
The application checks the cache first. On a miss, it reads from the database and populates the cache.

```rust
async fn get_user(cache: &Cache, db: &PgPool, id: UserId) -> Result<User, Error> {
    // 1. Check cache
    if let Some(user) = cache.get(&format!("user:{}", id)).await? {
        return Ok(user);
    }

    // 2. Cache miss — read from database
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id.0)
        .fetch_one(db)
        .await?;

    // 3. Populate cache
    cache.set(&format!("user:{}", id), &user, Duration::from_secs(300)).await?;

    Ok(user)
}
```

#### Write-Through
Every write goes to both the cache and the database simultaneously.
- **Pro:** Cache is always consistent with the database
- **Con:** Write latency increases (two writes per operation)

#### Write-Behind (Write-Back)
Writes go to the cache immediately. The cache asynchronously flushes to the database.
- **Pro:** Very fast writes
- **Con:** Data loss risk if the cache crashes before flushing

**Cache invalidation** is one of the hardest problems in computer science:

| Strategy | How it works | Trade-off |
|----------|-------------|-----------|
| **TTL (Time to Live)** | Cache entries expire after a fixed time | Simple but may serve stale data |
| **Event-based** | Invalidate when the data changes | Consistent but requires event infrastructure |
| **Version-based** | Include a version in the cache key | Never stale but requires version tracking |

### Message Queues & Event Streaming

#### Message Queues (RabbitMQ, SQS)
Point-to-point: a message is consumed by exactly one consumer. Used for task distribution.

```
Producer ──→ [Queue] ──→ Consumer 1
                    ──→ Consumer 2  (each message goes to ONE consumer)
                    ──→ Consumer 3
```

**Use cases:** Background jobs, email sending, image processing, order fulfillment.

#### Event Streaming (Kafka, Redpanda)
Pub/sub with persistence: events are published to topics and can be consumed by multiple independent consumer groups. Events are stored for a retention period (hours to indefinitely).

```
Producer ──→ [Topic: "orders"] ──→ Consumer Group A (Inventory)
                               ──→ Consumer Group B (Analytics)
                               ──→ Consumer Group C (Email)
```

**Key differences:**

| Feature | Message Queue | Event Streaming |
|---------|--------------|-----------------|
| **Delivery** | Each message to one consumer | Each event to all subscriber groups |
| **Persistence** | Deleted after consumption | Retained for replay |
| **Ordering** | FIFO within a queue | Ordered within a partition |
| **Replay** | No (message is gone) | Yes (consumers can re-read from any offset) |
| **Use case** | Task distribution | Event-driven architecture, audit logs |

### Microservices vs Monolith

**Monolith:** One deployable unit containing all functionality.

**Microservices:** Many small, independently deployable services, each owning a specific domain.

| Factor | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | All-or-nothing | Independent per service |
| **Scaling** | Entire application scales together | Scale individual services |
| **Development** | Simple at first, complex at scale | Complex setup, but teams are independent |
| **Data** | Shared database | Each service owns its data |
| **Debugging** | Stack trace tells the whole story | Distributed tracing across services |
| **Latency** | In-process calls (nanoseconds) | Network calls (milliseconds) |
| **Consistency** | ACID transactions | Eventual consistency (usually) |

**When to migrate to microservices:**
- Teams are stepping on each other in the monolith
- Different components need to scale independently (CPU-intensive search vs I/O-heavy order processing)
- Deployment of one component shouldn't risk breaking another
- You have the operational maturity (CI/CD per service, monitoring, distributed tracing)

### Service Mesh & API Gateways

**API Gateway:** A single entry point for all client requests. Handles routing, authentication, rate limiting, and request transformation.

```
Mobile App ──→ ┌──────────────┐ ──→ User Service
Web App   ──→ │  API Gateway  │ ──→ Order Service
Partner   ──→ └──────────────┘ ──→ Payment Service
```

**Service Mesh (Istio, Linkerd):** A dedicated infrastructure layer for service-to-service communication. Handles load balancing, retries, circuit breaking, mutual TLS, and observability — without changing application code.

```
[Service A] ←→ [Sidecar Proxy] ←→ [Sidecar Proxy] ←→ [Service B]
                    ↑                     ↑
                    └──── Control Plane ──┘
```

### System Design Case Studies

#### URL Shortener

**Requirements:** Shorten URLs, redirect short URLs, 100M URLs, 10K reads/sec.

```
┌─────────┐     ┌──────────┐     ┌────────────┐     ┌──────────┐
│  Client  │────→│   CDN    │────→│ API Server │────→│ Database │
│          │     │ (cached  │     │            │     │(Postgres)│
│          │←────│ redirects│←────│  + Cache   │←────│          │
└─────────┘     └──────────┘     │  (Redis)   │     └──────────┘
                                  └────────────┘
```

**Key decisions:**
- Short code generation: Base62 encoding of auto-increment ID or random generation with collision check
- Cache popular URLs in Redis (most redirects hit a small set of URLs)
- CDN for read-heavy redirect traffic
- Database for persistence and analytics

#### Chat System

**Requirements:** Real-time messaging, 1M concurrent users, message history.

```
┌─────────┐     ┌──────────────┐     ┌────────────┐
│  Client  │←──→│  WebSocket   │←──→ │  Message   │
│          │    │  Gateway     │     │  Service   │
└─────────┘     └──────────────┘     └─────┬──────┘
                                           │
                    ┌──────────────────────┤
                    ↓                      ↓
              ┌──────────┐          ┌──────────┐
              │  Kafka   │          │ Cassandra│
              │ (fanout) │          │ (history)│
              └──────────┘          └──────────┘
```

**Key decisions:**
- WebSocket connections for real-time delivery
- Kafka for message fanout (sender publishes once, all recipients' gateways consume)
- Cassandra for message storage (write-heavy, time-ordered, horizontal scaling)
- Presence service for online/offline status (Redis with TTL)

#### News Feed System

**Requirements:** Personalized feed, 500M users, mix of followed users' content.

**Two approaches:**

**Fan-out on write (push model):**
When a user posts, pre-compute the feed for all followers.
- Pro: Feed reads are instant (pre-computed)
- Con: Celebrity with 10M followers = 10M writes per post

**Fan-out on read (pull model):**
When a user opens their feed, compute it on the fly from followed users' posts.
- Pro: No wasted computation for inactive users
- Con: Slow feed generation (must query many users' posts)

**Hybrid (what Facebook/Instagram use):**
- Fan-out on write for most users (pre-compute)
- Fan-out on read for celebrities (compute on demand, cache aggressively)

### Back-of-the-Envelope Estimation

Quick calculations to determine if a design is feasible.

**Common numbers to know:**

| Resource | Value |
|----------|-------|
| L1 cache reference | 0.5 ns |
| Main memory reference | 100 ns |
| SSD random read | 150 μs |
| HDD seek | 10 ms |
| Network round trip (same datacenter) | 0.5 ms |
| Network round trip (cross-continent) | 150 ms |
| 1 GB memory | ~$5/month (cloud) |
| 1 TB SSD storage | ~$100/month (cloud) |

**Example estimation: "Can we store 1B URLs in memory?"**
- Average URL: 100 bytes
- Short code + metadata: 50 bytes
- Total per record: 150 bytes
- 1B records × 150 bytes = 150 GB
- A large cloud instance has 256-512 GB RAM → Yes, but tight. Use SSD + cache hot URLs.

## Business Value

- **Handling growth**: Proper system design enables the business to grow without hitting walls. A system that can't handle 10x traffic will fail when a marketing campaign succeeds.
- **Cost efficiency**: Right-sized architecture avoids both under-provisioning (outages) and over-provisioning (waste). Caching alone can reduce database costs by 80-90%.
- **Time-to-market**: Choosing the right architecture upfront avoids costly rearchitecting later. The cost of fixing a design flaw increases 10-100x once the system is in production.
- **Competitive advantage**: Systems that respond in 100ms vs 2 seconds have measurably higher conversion rates. Amazon found that every 100ms of latency cost them 1% in sales.
- **Reliability**: Distributed system design (redundancy, failover, load balancing) translates directly to uptime SLAs, which translate to customer trust and contractual obligations.

## Real-World Examples

### Twitter's Timeline Architecture
Twitter's timeline (feed) evolved through multiple architectures. Initially, fan-out on read (pull model) — too slow as the user base grew. They switched to fan-out on write (push model) — a tweet is written into every follower's timeline cache. For celebrities with 50M+ followers, they use a hybrid: fan-out on write for regular users, fan-out on read for celebrity tweets. This hybrid approach balances write efficiency with read performance.

### Uber's Real-Time Matching System
Uber's core system matches riders with drivers in real time. Key design: a geospatial index (Google S2 library) partitions the Earth into cells, with drivers indexed by current cell. When a rider requests a ride, the system queries nearby cells for available drivers, ranks them by ETA, and sends the closest. The system handles millions of location updates per second using a combination of in-memory processing and event streaming.

### WhatsApp's Architecture for 2B Users
WhatsApp serves 2 billion users with remarkably few servers (relative to their scale). Key decisions: Erlang/OTP for the messaging backend (excellent for concurrent connections), message delivery optimized for "store and forward" (messages go through the server but aren't stored long-term), and end-to-end encryption (the server never sees message content, simplifying storage). Their efficiency demonstrates that good architecture enables scale without massive infrastructure.

### Netflix's Content Delivery
Netflix serves 15% of global internet bandwidth. Their architecture: content is encoded into multiple formats/qualities, stored in Open Connect Appliances (custom servers) deployed at ISPs worldwide. When you press play, the client selects the nearest CDN node and the optimal quality stream. The control plane (recommendations, search, authentication) runs on AWS, while the data plane (actual video streaming) runs on their own CDN. This separation enables independent scaling of different concerns.

## Common Mistakes & Pitfalls

- **Designing for Google scale on day one** — Your startup doesn't need Kafka, Kubernetes, and a service mesh for 100 users. Start simple, measure, and scale where data tells you to.

- **Ignoring back-of-the-envelope math** — Not doing basic calculations. "Can we store this in memory?" and "How many requests per second?" should be answered before committing to an architecture.

- **Single points of failure** — A system is only as reliable as its weakest link. One database server, one load balancer, one DNS provider — each is a SPOF.

- **Caching without invalidation strategy** — Caching is easy. Cache invalidation is hard. If you can't answer "when does this cache entry become stale?", you have a bug waiting to happen.

- **Synchronous chains** — Service A calls B, which calls C, which calls D. If any service is slow, the entire chain is slow. Prefer async communication (events, queues) for non-time-critical operations.

- **Shared databases between services** — Two services reading/writing the same database tables creates tight coupling. Each service should own its data.

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **Cache everything** | Fast reads, reduced DB load | Stale data risk, memory cost, invalidation complexity |
| **Event-driven** | Loose coupling, resilient | Eventual consistency, harder debugging |
| **Synchronous APIs** | Simple, consistent, easy to reason about | Tight coupling, cascading failures |
| **Monolith** | Simple, fast to develop | Scales as one unit, deployment coupling |
| **Microservices** | Independent scaling, team autonomy | Operational complexity, network overhead |
| **CDN** | Fast for static/cacheable content | Cost at scale, cache invalidation delay |

## When to Use / When Not to Use

**Load balancer — always use when:**
- Running more than one server instance
- Need high availability (automatic failover)

**Caching — use when:**
- Read-heavy workloads (read:write ratio > 10:1)
- Data that's expensive to compute or fetch
- Data that's acceptable to be slightly stale

**Message queues — use when:**
- Background processing (email, image resize, reports)
- Decoupling producers from consumers
- Smoothing traffic spikes

**Event streaming — use when:**
- Multiple consumers need the same events
- Event replay is needed (reprocessing, new consumers)
- Building an event-driven architecture

**Microservices — use when:**
- You have 50+ engineers needing team independence
- Components have different scaling requirements
- You have operational maturity for distributed systems

## Key Takeaways

1. Scale vertically first — it's simpler. Scale horizontally only when you need to.
2. Caching is the highest-leverage performance optimization. Most applications are read-heavy, and caching makes reads nearly free.
3. Choose between message queues (point-to-point) and event streaming (pub/sub with replay) based on your consumption pattern.
4. Back-of-the-envelope estimation prevents impossible designs. Always do the math before committing to an architecture.
5. Start with a monolith, extract microservices when you have concrete scaling or organizational needs.
6. Every system design involves trade-offs. There is no perfect architecture — only appropriate ones for your constraints.
7. Study real-world architectures (Twitter, Uber, Netflix, WhatsApp). The best learning comes from understanding why they made their choices.

## Further Reading

- **Books:**
  - *Designing Data-Intensive Applications* — Martin Kleppmann (2017) — The essential guide to distributed system fundamentals
  - *System Design Interview* — Alex Xu (Vol 1 & 2) — Practical system design walkthroughs
  - *Building Microservices* — Sam Newman (2nd edition, 2021) — Comprehensive microservices guide

- **Papers & Articles:**
  - [System Design Primer](https://github.com/donnemartin/system-design-primer) — Open-source collection of system design resources
  - [High Scalability Blog](http://highscalability.com/) — Architecture case studies of major systems
  - [Netflix Tech Blog](https://netflixtechblog.com/) — Detailed posts on Netflix's architecture
  - [Uber Engineering Blog](https://www.uber.com/blog/engineering/) — System design at Uber's scale

- **Talks:**
  - *Scaling Instagram Infrastructure* — Lisa Guo (QCon) — How Instagram scales PostgreSQL
  - *The Evolution of Reddit's Architecture* — Neil Williams — From monolith to microservices
