# Rate Limiting & Backpressure

## Why Rate Limit?

Rate limiting protects your API from abuse and ensures fair resource distribution. Without it, a single misbehaving client — whether malicious or buggy — can consume all available resources and degrade the service for everyone.

Rate limiting is not optional. Every API needs it, including internal ones. A runaway batch job or a retry loop with no backoff can take down an internal service just as effectively as a DDoS attack.

## Rate Limiting Algorithms

### Token Bucket

The most widely used algorithm. A bucket holds tokens that refill at a fixed rate. Each request costs one (or more) tokens. If the bucket is empty, the request is rejected.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Time 0s:  Bucket has 100 tokens
           Client sends 50 requests → 50 tokens remain
Time 1s:  10 tokens refill → 60 tokens
           Client sends 80 requests → rejected (only 60 available)
Time 5s:  Bucket refills to 100 (capped at capacity)
           Client sends 100 requests → burst allowed, 0 tokens remain
```

**Properties:**
- Allows bursts up to the bucket capacity
- Sustains the refill rate over time
- Simple to implement and reason about
- Most APIs use this: Stripe, GitHub, AWS

### Sliding Window

Count requests in a rolling time window. If the count exceeds the limit, reject.

```
Limit: 100 requests per 60 seconds

Request at T=45s: Count requests from T=-15s to T=45s
  If count < 100 → allow
  If count >= 100 → reject
```

**Precise sliding window** tracks each request's timestamp. **Sliding window counter** approximates by weighting the previous and current fixed windows:

```
Previous window: 70 requests (40% of window remaining)
Current window:  20 requests (60% of window elapsed)
Estimated count: 70 * 0.4 + 20 = 48 → under limit, allow
```

Good for simple rate limits ("100 requests per minute"). Less burst-tolerant than token bucket.

### Fixed Window

Count requests in fixed time intervals (e.g., every minute from :00 to :59). Reset the counter at each boundary.

```
Window 12:00-12:01: 80/100 requests used
Window 12:01-12:02: counter resets to 0
```

**Problem:** A client can send 100 requests at 12:00:59 and 100 more at 12:01:00 — 200 requests in 2 seconds while technically respecting the 100/minute limit. This boundary burst problem makes fixed windows unsuitable for strict rate control.

### Leaky Bucket

Requests enter a queue (bucket) and are processed at a fixed rate. If the queue is full, new requests are dropped.

```
Queue capacity: 50
Drain rate: 10 requests/second

Incoming: 30 requests arrive at once
  → 30 enter the queue
  → processed at 10/sec (takes 3 seconds)
  → no rejection, but added latency

Incoming: 80 requests arrive at once
  → 50 enter the queue, 30 rejected immediately
  → processed at 10/sec (takes 5 seconds)
```

Produces a smooth, predictable outflow rate. Best for traffic shaping, but adds latency. Used more in networking than in API rate limiting.

## Rate Limit Headers

Standard response headers communicate rate limit status to clients:

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 73
X-RateLimit-Reset: 1710500000

# When rate limited:
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1710500000
```

**There is now an IETF draft standard** (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`) without the `X-` prefix. Adoption is growing.

**Best practice:** Always include rate limit headers in every response, not just 429 responses. Clients should be able to self-throttle before hitting the limit.

## Rate Limiting Scope

Rate limits can apply at different granularities:

| Scope | Example | Use case |
|-------|---------|----------|
| **Per API key** | 1000 req/min per key | Most common for public APIs |
| **Per user** | 100 req/min per authenticated user | Preventing abuse by individual users |
| **Per IP** | 50 req/min per IP | Unauthenticated endpoints, login |
| **Per endpoint** | `/search`: 10 req/min, `/users`: 100 req/min | Protecting expensive operations |
| **Global** | 10,000 req/min total | Protecting the backend from overload |

Stripe uses tiered rate limiting: 100 reads/sec in live mode, 25 reads/sec in test mode, with lower limits on specific expensive endpoints.

## Load Shedding

Load shedding goes beyond rate limiting: when the system is overloaded, reject requests immediately to protect the requests you can handle.

**Priority-based shedding:**
```
if system_load > 90%:
    reject low-priority requests (analytics, batch)
if system_load > 95%:
    reject medium-priority requests (reads)
if system_load > 99%:
    reject everything except health checks
```

**Random early detection:** As load increases, randomly reject an increasing percentage of requests. This prevents the thundering herd problem where all requests are rejected simultaneously.

**Circuit breaker pattern:** When a downstream dependency fails, stop sending requests to it entirely. After a timeout, send a probe request. If it succeeds, resume normal traffic.

## Backpressure

When a service can't keep up with incoming requests, it needs to signal callers to slow down — that's backpressure.

**Backpressure mechanisms:**
- **HTTP 429** — Tell the client to back off
- **Load shedding** — Reject excess requests immediately (fail fast)
- **Queue with bounded size** — Accept requests into a queue, reject when full
- **Streaming flow control** — gRPC/HTTP/2 built-in flow control

**Without backpressure:** The service queues requests unboundedly, memory grows, latency increases for everyone, and the system crashes.

**With backpressure:** Excess requests are rejected fast, callers retry with backoff, and the service stays healthy for requests it can handle.

### Client-Side Backoff

Clients must cooperate with backpressure signals:

```
Exponential backoff with jitter:
  Attempt 1: wait 1s + random(0, 0.5s)
  Attempt 2: wait 2s + random(0, 1.0s)
  Attempt 3: wait 4s + random(0, 2.0s)
  Attempt 4: wait 8s + random(0, 4.0s)
  ... cap at 60s
```

**Jitter is critical.** Without it, all clients retry at the same time after a rate limit window resets, causing a thundering herd.

## Rust Implementation

### Token Bucket Rate Limiter

```text
STRUCTURE TokenBucket:
    capacity ← integer
    tokens ← float
    refill_rate ← float  // tokens per second
    last_refill ← timestamp

PROCEDURE NEW_TOKEN_BUCKET(capacity, refill_rate):
    RETURN TokenBucket {
        capacity ← capacity,
        tokens ← capacity,
        refill_rate ← refill_rate,
        last_refill ← NOW()
    }

PROCEDURE TRY_ACQUIRE(bucket, cost):
    REFILL(bucket)
    IF bucket.tokens ≥ cost THEN
        bucket.tokens ← bucket.tokens - cost
        RETURN true
    ELSE
        RETURN false

PROCEDURE REMAINING(bucket):
    RETURN FLOOR(bucket.tokens)

PROCEDURE REFILL(bucket):
    now ← NOW()
    elapsed ← SECONDS_SINCE(bucket.last_refill, now)
    bucket.tokens ← MIN(bucket.tokens + elapsed * bucket.refill_rate, bucket.capacity)
    bucket.last_refill ← now
```

### axum Rate Limiting Middleware

```text
// Rate limit store: shared map of client key → TokenBucket

PROCEDURE RATE_LIMIT_MIDDLEWARE(client_address, request, next):
    store ← GET rate limit store FROM request extensions
    key ← client_address.ip AS string

    LOCK store
    bucket ← LOOKUP key IN store, OR INSERT NEW TokenBucket(capacity=100, refill_rate=10.0)

    IF TRY_ACQUIRE(bucket, 1) THEN
        remaining ← REMAINING(bucket)
        UNLOCK store  // Release lock before processing

        response ← FORWARD request TO next handler

        // Add rate limit headers to every response
        SET response header "X-RateLimit-Limit" ← "100"
        SET response header "X-RateLimit-Remaining" ← remaining

        RETURN response
    ELSE
        UNLOCK store

        SET headers:
            "Retry-After" ← "10"
            "X-RateLimit-Limit" ← "100"
            "X-RateLimit-Remaining" ← "0"

        RETURN (429 TOO MANY REQUESTS, headers, "Rate limit exceeded")
```

### Distributed Rate Limiting with Redis

For multi-instance deployments, rate limiting must be centralized. Redis is the standard choice:

```text
// Sliding window rate limit using Redis
// Lua script for atomicity
REDIS_SCRIPT RATE_LIMIT_SCRIPT:
    key ← KEYS[1]
    limit ← TO_NUMBER(ARGV[1])
    window ← TO_NUMBER(ARGV[2])
    now ← TO_NUMBER(ARGV[3])

    // Remove expired entries
    REDIS ZREMRANGEBYSCORE key, 0, now - window

    // Count current requests
    count ← REDIS ZCARD key

    IF count < limit THEN
        // Add current request
        REDIS ZADD key, now, now + "-" + RANDOM(1000000)
        REDIS EXPIRE key, window
        RETURN {1, limit - count - 1}  // allowed, remaining
    ELSE
        RETURN {0, 0}  // rejected, 0 remaining
```

This uses a Redis sorted set where the score is the request timestamp. Entries outside the window are removed, and the set's cardinality gives the request count. The Lua script ensures atomicity.

## Common Mistakes

- **No rate limiting at all** — The most common mistake. Even internal APIs need it.
- **Rate limiting only at the edge** — If the API gateway rate limits but internal services don't, a compromised or buggy internal client can still cause cascading failures.
- **No `Retry-After` header** — Clients can't cooperate if you don't tell them when to retry.
- **Fixed backoff without jitter** — Causes thundering herds after rate limit windows reset.
- **Unbounded queues** — Accepting all requests into a queue without a size limit is just deferred failure with worse latency.
- **Ignoring the 429 response** — Client-side: always respect `Retry-After`. Never retry immediately.
