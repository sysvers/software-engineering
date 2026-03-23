# Scaling Fundamentals

## What is Scaling?

Scaling is the ability of a system to handle increasing load — more users, more data, more transactions — without degrading performance. Every successful product will eventually face a scaling challenge. The question is not *if* but *when*, and the architecture you choose determines how painful that moment will be.

## Vertical Scaling (Scale Up)

Add more resources to a single machine: more CPU cores, RAM, faster SSDs, better network cards.

**How it works:** You take your existing server, shut it down (or live-migrate), upgrade the hardware or move to a bigger instance type, and restart. Your application code does not change at all.

**Advantages:**
- Zero code changes — your application runs exactly as before
- No distributed system complexity — no network partitions, no split-brain, no eventual consistency
- Simple operations — one server to monitor, back up, and maintain
- Strong consistency — a single database on a single machine gives you ACID guarantees trivially

**Disadvantages:**
- Hard physical limits — the largest cloud instances top out around 400 vCPUs and 24 TB RAM (AWS u-24tb1.metal). Beyond that, there is no bigger box
- Single point of failure — if that one machine dies, everything dies
- Expensive at the top — the largest instances cost disproportionately more per unit of compute. A machine with 2x the CPU often costs 2.5-3x the price
- Downtime during upgrades — unless your cloud provider supports live migration, scaling up means a restart

**When vertical scaling is the right call:**
- Your application is early-stage and serves hundreds to low thousands of users
- Your workload is inherently single-threaded or hard to parallelize (some scientific computations, certain database queries)
- You want to delay the operational complexity of distributed systems

### Real-World Example: Stack Overflow

Stack Overflow serves over 100 million monthly visitors with remarkably few servers. As of their published architecture, their entire production setup runs on about 9 web servers and 4 SQL servers — all vertically scaled with high-end hardware (beefy CPUs, 384+ GB RAM, NVMe SSDs). They chose vertical scaling because:
- Their read-heavy workload fits comfortably in memory caches
- SQL Server on a powerful machine handles their query load
- Fewer servers mean dramatically simpler operations

They delayed horizontal scaling for years because a single well-tuned server handled far more than most engineers expect.

## Horizontal Scaling (Scale Out)

Add more machines and distribute the workload across them.

**How it works:** Instead of making one machine bigger, you run many copies of your application behind a load balancer. Data is partitioned (sharded) across multiple database servers. Stateless services are replicated freely.

**Advantages:**
- Near-infinite scaling — need more capacity? Add more machines
- Fault tolerance — one machine dying does not take down the system (if designed correctly)
- Cost efficiency at scale — commodity hardware is cheaper per unit of compute than high-end machines
- Geographic distribution — you can place machines closer to users in different regions

**Disadvantages:**
- Distributed system complexity — you now deal with network partitions, clock skew, partial failures, and eventual consistency
- Data consistency is hard — a write to one database replica may not be immediately visible on another
- Operational overhead — more machines means more monitoring, more deployment targets, more things that can break
- Application changes required — your code must handle distributed state, session management, and idempotent operations

**When horizontal scaling is necessary:**
- You have exhausted practical vertical scaling limits
- You need high availability (no single point of failure)
- Different components have different resource needs (CPU-bound vs I/O-bound)
- You serve users globally and need low latency in multiple regions

### Real-World Example: Instagram's Database Sharding

Instagram hit the limits of a single PostgreSQL instance around 2012 when their user growth exploded. Their solution: shard the database horizontally by user ID. Each shard is a PostgreSQL instance holding a range of user IDs. This let them scale to billions of photos and hundreds of millions of users. The cost: cross-shard queries became expensive, and features like "global search" required separate infrastructure (Elasticsearch).

## The Practical Scaling Path

The industry consensus, validated by companies like Shopify, GitHub, and Basecamp:

1. **Start with a monolith on a single server.** Get your product right first.
2. **Scale vertically.** Upgrade to a bigger machine. This buys you more time than you think.
3. **Add caching.** Redis or Memcached in front of your database can reduce load by 80-90%.
4. **Add read replicas.** Separate read traffic from write traffic.
5. **Scale horizontally** only when the above is insufficient. Shard the database, add more application servers behind a load balancer.

Most applications never need to go past step 3. The myth that you need microservices and Kubernetes from day one has caused more engineering pain than it has solved.

## Back-of-the-Envelope Estimation

Before choosing an architecture, do the math. Quick calculations reveal whether a design is feasible or absurd.

### Numbers Every Engineer Should Know

| Resource | Latency / Cost |
|----------|---------------|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Main memory reference | 100 ns |
| SSD random read | 150 us |
| HDD seek | 10 ms |
| Network round trip (same datacenter) | 0.5 ms |
| Network round trip (cross-continent) | 150 ms |
| Read 1 MB sequentially from memory | 250 us |
| Read 1 MB sequentially from SSD | 1 ms |
| Read 1 MB sequentially from HDD | 20 ms |
| 1 GB memory (cloud) | ~$5/month |
| 1 TB SSD storage (cloud) | ~$100/month |

### Capacity Planning Walkthrough

**Example: "Can we store 1 billion URLs in memory?"**
- Average URL: 100 bytes
- Short code + metadata: 50 bytes
- Total per record: 150 bytes
- 1B records x 150 bytes = 150 GB
- A large cloud instance has 256-512 GB RAM
- Verdict: Yes, but tight. Better to use SSD + cache the hot URLs in memory.

**Example: "Can a single server handle 10,000 requests per second?"**
- A modern server with an async runtime (Tokio in Rust, epoll in C) can handle 50,000-100,000+ simple HTTP requests per second
- If each request does a database query taking 5ms, you need 50 concurrent connections to sustain 10K req/s
- A connection pool of 50 on a single PostgreSQL instance is very reasonable
- Verdict: Yes, easily — if the requests are simple. If each request involves complex computation or multiple database queries, you will need to profile and measure.

**Example: "How much bandwidth for 1M daily active video watchers?"**
- Average session: 30 minutes of 1080p video at 5 Mbps
- Per user per day: 30 min x 60 sec x 5 Mbps = 9 Gb = ~1.1 GB
- 1M users x 1.1 GB = 1.1 PB per day
- Peak traffic (assume 3x average): ~300 Gbps
- Verdict: You need a CDN. No single datacenter can serve this economically.

## Capacity Planning in Practice

Capacity planning is not a one-time exercise. It is an ongoing process:

1. **Measure current usage** — instrument your system to know CPU, memory, disk I/O, network, and request rates
2. **Project growth** — based on business forecasts (new markets, product launches, seasonal peaks)
3. **Identify bottlenecks** — which resource will hit its limit first? That is your scaling priority
4. **Plan headroom** — maintain at least 30-50% headroom on critical resources. When you are at 70% capacity, start working on the next scaling step
5. **Load test regularly** — synthetic load tests reveal bottlenecks before real users hit them

### Real-World Example: Amazon Prime Day

Amazon plans for Prime Day months in advance. They run "GameDay" exercises — simulated traffic spikes — to validate that their scaling assumptions hold. In 2022, Prime Day handled 300 million items purchased in 48 hours. This required pre-provisioning compute capacity, pre-warming caches, and ensuring every service could handle the projected peak independently. The lesson: capacity planning is a team sport that spans engineering, product, and business.

## Common Pitfalls

- **Designing for Google scale on day one.** Your startup does not need Kafka, Kubernetes, and a service mesh for 100 users. Start simple, measure, and scale where data tells you to.
- **Ignoring the math.** Not doing basic calculations before committing to an architecture leads to systems that are either wildly over-provisioned (expensive) or under-provisioned (outages).
- **Scaling the wrong thing.** Adding more application servers when the database is the bottleneck does nothing. Always identify the actual bottleneck first.
- **Forgetting about data gravity.** Data is hard to move. Once you have 10 TB in one region, migrating to another is a multi-week project. Plan your data placement early.
- **Premature sharding.** Sharding a database is a one-way door. It is extremely difficult to un-shard. Exhaust all other options (vertical scaling, read replicas, caching) before sharding.

## Key Takeaways

1. Scale vertically first — it is simpler, cheaper, and buys more time than most engineers expect.
2. Always do back-of-the-envelope math before choosing an architecture.
3. Most applications can serve thousands of concurrent users on a single well-optimized server.
4. Horizontal scaling introduces distributed system complexity that should not be adopted lightly.
5. Capacity planning is continuous — measure, project, and plan headroom.
