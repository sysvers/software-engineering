# Distributed Caching

## Overview

Caching in distributed systems reduces latency and load on backend stores by keeping frequently accessed data in fast, in-memory storage closer to the application. The challenge is not just caching itself -- it is maintaining consistency across multiple cache nodes and invalidating stale data reliably in a system where failures and network partitions are normal.

---

## Cache Topologies

### Local (In-Process) Cache

Each application instance maintains its own cache in memory (e.g., a `HashMap` with TTL).

- Fastest access (no network hop)
- No consistency between instances -- each has its own copy
- Limited by the application's memory
- Good for: immutable data, configuration, short-lived request-scoped caches

### Centralized Cache

A dedicated cache cluster (e.g., Redis, Memcached) shared by all application instances.

- Single source of truth for cached data
- Network round-trip on every access (typically sub-millisecond on a local network)
- The cache cluster itself must be scaled and made highly available
- Good for: shared session state, frequently queried database results, rate limiting counters

### Two-Tier (Local + Remote)

Application instances maintain a small local L1 cache backed by a shared remote L2 cache.

- L1 handles the hottest data with zero network cost
- L2 prevents redundant database queries across instances
- Invalidation must propagate to both tiers
- Good for: high-throughput systems where even cache-hit latency matters

---

## Caching Patterns

### Cache-Aside (Lazy Loading)

The application checks the cache first. On a miss, it reads from the database, populates the cache, and returns the result.

```text
STRUCTURE CacheEntry<V>
    value : V
    inserted_at : Timestamp
    ttl : Duration

PROCEDURE CacheEntry.IS_EXPIRED() → Boolean
    RETURN ELAPSED(self.inserted_at) > self.ttl

/// A cache-aside implementation with TTL-based expiration.
STRUCTURE CacheAside<V>
    store : Map<String, CacheEntry<V>>
    default_ttl : Duration

/// Attempt to read from cache. Returns NULL on miss or expiration.
PROCEDURE CacheAside.GET(key) → Optional<V>
    entry ← self.store[key]
    IF entry IS NOT NULL THEN
        IF NOT entry.IS_EXPIRED() THEN
            RETURN entry.value
        END IF
        DELETE self.store[key]
    END IF
    RETURN NULL

/// Populate the cache after a database read.
PROCEDURE CacheAside.SET(key, value)
    self.store[key] ← CacheEntry {
        value ← value,
        inserted_at ← CURRENT_TIME(),
        ttl ← self.default_ttl
    }

/// Invalidate a cache entry, typically after a write to the database.
PROCEDURE CacheAside.INVALIDATE(key)
    DELETE self.store[key]
```

- Most common pattern in practice
- Risk of thundering herd: many clients simultaneously miss and all query the database
- Mitigation: use a "single-flight" or lock mechanism so only one client fetches on a miss

### Write-Through

Every write goes to both the cache and the database atomically.

- Ensures the cache is always consistent with the database
- Higher write latency (two writes on every mutation)
- Good for: read-heavy workloads where consistency matters and writes are infrequent

### Write-Behind (Write-Back)

Writes go to the cache first and are asynchronously flushed to the database in batches.

- Very low write latency
- Risk of data loss if the cache node fails before flushing
- Complexity in ensuring ordering and handling flush failures
- Good for: write-heavy workloads that can tolerate eventual durability (e.g., view counters, analytics)

---

## Cache Invalidation Across Nodes

Cache invalidation is famously one of the two hard problems in computer science. In a distributed system, it gets harder because invalidation must propagate across multiple cache nodes and application instances.

### Strategies

**TTL-based expiration:** Every entry has a time-to-live. After it expires, the next read triggers a fresh fetch. Simple and reliable but allows stale reads up to the TTL duration.

**Event-driven invalidation:** When data changes, the writing service publishes an event (e.g., via Kafka, Redis Pub/Sub). Subscribers invalidate or update their caches. Lower staleness than TTL but adds messaging infrastructure.

**Lease-based invalidation (Facebook's approach):** The cache issues a "lease" (a token) when a client gets a miss. The client can only write back to the cache if the lease is still valid. If another client invalidates the key while the first is fetching, the lease is revoked and the stale write is rejected.

**Version-stamped values:** Each cached value carries a version number. Writers increment the version. Readers accept only values with a version >= what they expect. Prevents older values from overwriting newer ones.

---

## Redis Cluster

Redis Cluster is the most widely used distributed caching system in modern architectures.

**Architecture:**
- Data is split across 16,384 hash slots
- Each master node owns a subset of hash slots
- Each master can have one or more replicas for failover
- Clients are hash-slot-aware: they compute `CRC16(key) mod 16384` and route directly to the correct node

**Key behaviors:**
- **No multi-key operations across slots** (unless keys are on the same node via hash tags: `{user:123}.profile` and `{user:123}.settings` share a slot)
- **Gossip protocol** for cluster state propagation
- **Automatic failover** when a master fails, a replica is promoted
- **MOVED and ASK redirects** handle rebalancing transparently to clients

---

## Real-World Examples

### Facebook Memcache (TAO and Memcached at Scale)

Facebook operates one of the largest Memcached deployments in the world, serving billions of requests per second. Key insights from their published architecture:

- **Regional pools:** Memcached instances are deployed in regional clusters. Reads are served locally; writes go to the primary region and are replicated.
- **Lease mechanism:** To prevent thundering herd and stale sets, Facebook introduced leases. When a cache miss occurs, the client receives a lease token. If the key is invalidated before the client returns with the data, the lease is revoked and the stale data is not written.
- **Gutter pools:** Dedicated "gutter" Memcached instances absorb load when primary cache instances fail. Rather than sending all misses to the database, failures are redirected to gutter servers with short TTLs.
- **McRouter:** A Memcached protocol-compatible routing layer that handles consistent hashing, replication, failover, and connection pooling. Applications talk to McRouter as if it were a single Memcached instance.
- **Delete-based invalidation:** On writes, Facebook deletes the cache key rather than updating it. This is simpler and avoids race conditions where an update with stale data overwrites a newer value.

### Netflix EVCache

Netflix's EVCache is a distributed caching solution built on top of Memcached, designed for their AWS infrastructure:

- **Zone-aware replication:** Data is replicated across AWS availability zones. Reads are served from the local zone; if the local cache is down, the client falls back to another zone.
- **Write-all, read-local:** On writes, EVCache writes to all zones. On reads, it only reads from the local zone. This ensures low read latency while maintaining cross-zone durability.
- **Tunable consistency:** Applications can choose to read from one zone (fast, potentially stale) or all zones (consistent but slower).
- **Near-cache (L1):** For extremely hot keys, EVCache supports an in-process L1 cache in front of the Memcached L2. This eliminates network hops for the most frequently accessed data.
- **Integration with Chaos Monkey:** EVCache is tested continuously with fault injection. Cache node failures are expected and handled gracefully by fallback to other zones.

---

## Common Pitfalls

1. **Cache stampede / thundering herd:** When a popular key expires, many clients simultaneously miss and all query the database. Use leases, single-flight patterns, or staggered TTLs with jitter.

2. **Inconsistency between cache and database:** Race conditions between write and invalidation paths can leave stale data in the cache. Prefer delete-on-write over update-on-write to reduce the window for races.

3. **Unbounded cache growth:** Without eviction policies (LRU, LFU, TTL), caches grow until they consume all available memory. Always configure max memory limits and an eviction policy.

4. **Caching negative results (or not):** If you do not cache misses (negative caching), repeated queries for nonexistent keys always hit the database. Cache misses with a short TTL to prevent this.

5. **Hot key problem:** A single key accessed millions of times per second can overwhelm a single cache node. Solutions include replicating hot keys across nodes, using local caches for hot keys, or splitting the key into sub-keys.

---

## Key Takeaways

- Cache-aside is the right default pattern. Use write-through only when consistency is critical, and write-behind only when you can tolerate eventual durability.
- Invalidation is harder than caching. Design your invalidation strategy before you start caching. Prefer delete over update for safer semantics.
- Two-tier caching (local L1 + remote L2) gives the best of both worlds for high-throughput systems but adds invalidation complexity.
- Redis Cluster is the dominant choice for new systems. Memcached remains relevant at Facebook/Meta scale and for simpler use cases.
- Always plan for cache failure. The system must function (perhaps at degraded performance) when the cache is unavailable.
