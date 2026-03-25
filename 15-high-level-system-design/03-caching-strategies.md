# Caching Strategies

## Why Caching Matters

Caching stores frequently accessed data closer to the consumer, reducing latency and load on backend systems. It is the single highest-leverage performance optimization for most applications, because most workloads are read-heavy (read:write ratios of 100:1 or higher are common).

A single Redis instance can serve 100,000+ read operations per second with sub-millisecond latency. Compare that to a database query that takes 5-50ms. Caching turns expensive operations into nearly free ones.

## Cache Levels

Data passes through multiple cache layers on its way to the user:

```
Client → [Browser Cache] → [CDN] → [API Gateway Cache] → [Application Cache] → [Database Cache] → [Database]
         Fastest                                                                                      Slowest
```

Each layer trades freshness for speed:
- **Browser cache:** Instant (no network call), but only for that single user
- **CDN:** 1-50ms (nearby edge server), shared across all users in a region
- **Application cache (Redis/Memcached):** 1-5ms, shared across all application servers
- **Database cache (query cache, buffer pool):** Transparent to the application, managed by the database engine

## Caching Patterns

### Cache-Aside (Lazy Loading)

The application manages the cache explicitly. On a read, check the cache first. On a miss, read from the database, then populate the cache.

```
1. App checks cache  →  Cache HIT  →  Return cached data
                     →  Cache MISS →  2. Read from DB
                                      3. Write to cache
                                      4. Return data
```

**Advantages:**
- Only requested data is cached (no wasted memory on unused data)
- Cache failure is not fatal — the application falls back to the database
- Simple to implement and reason about

**Disadvantages:**
- First request for any data is always slow (cache miss)
- Stale data risk — if the database is updated, the cache still holds the old value until TTL expires or explicit invalidation

```text
PROCEDURE GET_USER(cache, db, user_id):
    cache_key ← "user:" + user_id

    // Step 1: Check cache
    user ← AWAIT cache.GET(cache_key)
    IF user IS present THEN
        RETURN user  // Cache hit

    // Step 2: Cache miss -- fetch from database
    user ← AWAIT QUERY db: "SELECT * FROM users WHERE id = user_id"

    // Step 3: Populate cache with TTL
    AWAIT cache.SET(cache_key, user, TTL ← 300 seconds)

    RETURN user
```

### Write-Through

Every write goes to both the cache and the database simultaneously. The write is only considered successful when both complete.

```
App writes → Cache + Database (simultaneously)
App reads  → Cache (always populated)
```

**Advantages:**
- Cache is always consistent with the database (no stale reads)
- Reads are always fast (data is in the cache from the moment it is written)

**Disadvantages:**
- Write latency increases — every write does two operations
- Cache is filled with data that may never be read (wasted memory)
- More complex write path

**Best for:** Data that is read immediately after writing (e.g., user profile updates that the user sees right away).

### Write-Behind (Write-Back)

Writes go to the cache immediately. The cache asynchronously flushes to the database in batches.

```
App writes → Cache (immediate)  → [Async flush] → Database (eventually)
```

**Advantages:**
- Extremely fast writes (only writing to in-memory cache)
- Batch writes to the database reduce I/O
- Can absorb write spikes that would overwhelm the database

**Disadvantages:**
- Data loss risk — if the cache crashes before flushing, un-persisted writes are lost
- Complexity — need a reliable flush mechanism with retries and monitoring
- Inconsistency window — the database lags behind the cache

**Best for:** High-write-throughput scenarios where some data loss is acceptable (analytics counters, metrics, non-critical logs).

### Read-Through

The cache itself is responsible for loading data on a miss. The application always reads from the cache and never directly from the database.

```
App reads → Cache → (on miss) Cache reads from DB → Returns to App
```

**Advantages:**
- Simpler application code — the cache handles the loading logic
- Consistent cache population strategy

**Disadvantages:**
- Requires a cache library or proxy that supports read-through (e.g., Caffeine in Java, custom implementation in Rust)
- First-request latency is still slow (cache miss triggers DB read)

## Cache Invalidation

Cache invalidation is famously one of the two hard problems in computer science (the other being naming things). The core challenge: when the underlying data changes, how do you ensure the cache reflects the change?

### TTL (Time to Live)

Each cache entry expires after a fixed duration. After expiration, the next read triggers a fresh fetch from the database.

| TTL Value | Trade-off |
|-----------|-----------|
| Short (5-30s) | Near-fresh data, more database load |
| Medium (1-5min) | Good balance for most use cases |
| Long (1hr+) | Minimal DB load, but data can be very stale |

**Best for:** Data where some staleness is acceptable (user profiles, product catalogs, configuration).

### Event-Based Invalidation

When data changes, an event is emitted (via message queue, database trigger, or application code) that explicitly invalidates the cache entry.

```
1. App updates DB
2. App publishes "user:123:updated" event
3. Cache service receives event and deletes cache key "user:123"
4. Next read triggers a fresh cache-aside load
```

**Best for:** Data that must be consistent (pricing, inventory counts, permissions).

**Caveat:** Requires event infrastructure (Kafka, Redis Pub/Sub, database CDC). If the invalidation event is lost or delayed, the cache serves stale data.

### Version-Based Invalidation

Include a version identifier in the cache key. When the data changes, increment the version. Old cache entries naturally become unreachable (and eventually expire via TTL).

```
Cache key: "user:123:v7"  →  After update: "user:123:v8"
```

**Best for:** Static assets (CSS, JavaScript — this is how cache-busting with content hashes works) and data with a natural version (document revisions).

## CDN (Content Delivery Network)

A CDN caches content at edge servers geographically close to users. When a user in Tokyo requests an image, it is served from a Tokyo edge server (5ms) instead of the origin server in Virginia (150ms).

**What CDNs cache well:**
- Static assets (images, CSS, JavaScript, fonts)
- Video content (Netflix, YouTube)
- API responses that are the same for all users (public product data)

**What CDNs do NOT cache well:**
- Personalized content (user-specific feeds)
- Rapidly changing data (real-time dashboards)
- Authenticated API responses (unless using edge compute)

**CDN invalidation:** When you deploy new static assets, you need to invalidate the CDN cache. Strategies:
- **Cache-busting filenames:** `app.abc123.js` — each build produces a new filename, so the old cached version is never requested
- **Purge API:** Most CDNs offer an API to explicitly purge cached objects
- **Short TTL + stale-while-revalidate:** Serve stale content while fetching fresh content in the background

### Real-World Example: Netflix's CDN (Open Connect)

Netflix does not rely on third-party CDNs for video delivery. They built Open Connect — custom hardware appliances deployed inside ISP networks worldwide. During off-peak hours, popular content is pre-positioned on these appliances. When you press play, the client connects to the nearest appliance. This architecture serves 15% of global internet bandwidth. The key insight: for video at Netflix's scale, owning the CDN is cheaper and more reliable than renting.

## Rust TTL Cache Implementation

A thread-safe in-memory cache with per-entry TTL, suitable for application-level caching when you want to avoid an external dependency like Redis.

```text
/// A cache entry holding a value and its expiration timestamp.
STRUCTURE CacheEntry:
    value ← V
    expires_at ← timestamp

/// Thread-safe in-memory cache with per-entry TTL.
///
/// Design decisions:
/// - RwLock instead of Mutex: allows concurrent readers (reads are far
///   more frequent than writes in a cache).
/// - Lazy expiration: expired entries are removed on access, not by a
///   background thread. This avoids the complexity of a timer thread.
/// - Clone bound on V: entries are cloned on read to avoid holding the
///   lock across the caller's processing.
STRUCTURE TtlCache:
    entries ← shared read-write-locked map of Key → CacheEntry
    default_ttl ← duration

/// Create a new cache with a default TTL for entries.
PROCEDURE NEW_TTL_CACHE(default_ttl):
    RETURN TtlCache {
        entries ← NEW shared map,
        default_ttl ← default_ttl
    }

/// Insert a value with the default TTL.
PROCEDURE SET(cache, key, value):
    SET_WITH_TTL(cache, key, value, cache.default_ttl)

/// Insert a value with a custom TTL.
PROCEDURE SET_WITH_TTL(cache, key, value, ttl):
    ACQUIRE write lock ON cache.entries
    INSERT (key → CacheEntry { value, expires_at ← NOW() + ttl })

/// Get a value if it exists and has not expired.
/// Returns None for both missing and expired entries.
PROCEDURE GET(cache, key):
    // Try a read lock first (concurrent reads)
    ACQUIRE read lock ON cache.entries
    IF key EXISTS in map THEN
        entry ← map[key]
        IF NOW() < entry.expires_at THEN
            RETURN CLONE(entry.value)
        // Entry is expired -- fall through
    ELSE
        RETURN None  // Key does not exist

    // Entry exists but is expired -- acquire write lock to remove it
    ACQUIRE write lock ON cache.entries
    REMOVE key FROM map
    RETURN None

/// Delete a specific key (explicit invalidation).
PROCEDURE INVALIDATE(cache, key):
    ACQUIRE write lock ON cache.entries
    REMOVE key FROM map

/// Remove all expired entries. Call periodically to reclaim memory.
/// In production, run this on a timer (e.g., every 60 seconds).
PROCEDURE EVICT_EXPIRED(cache):
    ACQUIRE write lock ON cache.entries
    now ← NOW()
    RETAIN only entries WHERE now < entry.expires_at

/// Return the number of entries (including expired ones not yet evicted).
PROCEDURE LEN(cache):
    ACQUIRE read lock ON cache.entries
    RETURN SIZE OF map

/// Check if the cache is empty.
PROCEDURE IS_EMPTY(cache):
    ACQUIRE read lock ON cache.entries
    RETURN map IS empty

// Usage example: cache-aside pattern with the TtlCache
PROCEDURE GET_PRODUCT(cache, db, product_id):
    key ← "product:" + product_id

    // 1. Check cache
    cached ← GET(cache, key)
    IF cached IS present THEN
        RETURN cached

    // 2. Cache miss -- fetch from database
    product ← db.FETCH_PRODUCT(product_id)

    // 3. Populate cache
    SET(cache, key, CLONE(product))

    RETURN product
```

**Production considerations for this cache:**
- **Memory limits:** This implementation grows unboundedly. In production, add a max capacity with LRU eviction (remove the least recently used entry when full).
- **Thundering herd:** If a popular key expires, many concurrent requests will all miss the cache and hit the database simultaneously. Mitigation: use a lock per key so only one request fetches from the DB while others wait.
- **Serialization:** For a distributed cache (Redis), you would serialize values to bytes. For an in-memory cache like this, Clone is sufficient.

## Real-World Caching Examples

### Instagram's Caching Architecture

Instagram uses Memcached extensively. Their primary cache pattern is cache-aside with TTL. Key insights from their published architecture:
- They run a massive Memcached fleet (hundreds of servers) that caches database query results
- Cache hit rates exceed 99% for most data types — meaning only 1 in 100 requests actually hits the database
- They use lease-based invalidation to prevent thundering herd: when a key is missing, the first client gets a "lease" (permission to fetch and populate), and subsequent clients wait for the lease holder to populate the cache
- Stale reads are acceptable for most content (a like count being 2 seconds behind is fine)

### Facebook's TAO Cache

Facebook built TAO (The Associations and Objects cache), a geographically distributed cache for their social graph. Key design:
- Two-level caching: a local leader cache in each region and a remote master cache in the primary region
- Read requests go to the local leader (fast). Writes go to the master and are asynchronously replicated to leaders
- This accepts eventual consistency (a post may appear to friends in another region 1-2 seconds later) in exchange for read performance
- TAO handles billions of reads per second across Facebook's social graph

### Redis at Twitter

Twitter uses Redis for caching timelines. When you open Twitter, your home timeline is served from a pre-computed Redis cache. The timeline service pre-computes feeds using fan-out on write (for most users) and caches them in Redis sorted sets ordered by time. Cache misses (for inactive users whose cached timeline expired) trigger on-demand computation.

## Common Pitfalls

- **Caching without an invalidation strategy.** If you cannot answer "when does this cache entry become stale?", you have a bug waiting to happen. Every cached value needs a defined expiration or invalidation path.
- **Cache stampede (thundering herd).** A popular key expires, and 10,000 concurrent requests all miss the cache and hammer the database. Solutions: lock-based population, probabilistic early expiration, or stale-while-revalidate.
- **Caching errors.** If a database query fails and you cache the error (or null), all subsequent requests get the cached error. Always check for valid data before caching.
- **Unbounded cache growth.** An in-memory cache without size limits will eventually OOM the process. Always set a max size with an eviction policy (LRU, LFU).
- **Over-caching.** Caching data that is rarely read or changes frequently wastes memory and adds invalidation complexity. Cache what is hot and expensive to compute.
- **Not monitoring cache hit rates.** A cache with a 50% hit rate is doing half its job. Monitor hit rates, miss rates, and eviction rates. Aim for 95%+ hit rates on your primary caches.

## Key Takeaways

1. Cache-aside with TTL is the most common and safest starting pattern.
2. Write-through when you need strong consistency; write-behind when you need write throughput.
3. Cache invalidation is the hard part. Choose TTL for simplicity, event-based for consistency.
4. CDNs are essential for static content and geographically distributed users.
5. Monitor cache hit rates religiously — they tell you whether caching is actually helping.
6. Protect against thundering herd on popular keys.
