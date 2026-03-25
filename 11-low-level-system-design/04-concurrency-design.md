# Concurrency Design

Designing for concurrent access is critical in any system handling multiple requests simultaneously. Rust's ownership system prevents data races at compile time, but you still need to choose the right concurrency strategy for your use case.

## Shared-Nothing Architecture

The simplest and most scalable approach: each request handler gets its own copy of the data it needs. No shared mutable state.

```text
// Each request gets its own connection from the pool
ASYNC FUNCTION HANDLE_REQUEST(pool: PgPool) → Result<Response, Error>
    conn ← AWAIT pool.ACQUIRE()?    // Gets a connection, doesn't share it
    data ← AWAIT QUERY(conn)?
    RETURN Response.JSON(data)
```

The connection pool itself is shared, but each connection is exclusively owned by one handler. No locking, no contention, no data races.

**When shared-nothing works:** Most web request handlers. Each request reads from the database, computes a response, and returns. No state persists between requests. This should be your default approach.

**When it breaks down:** When you need in-memory state that multiple requests read or write: caches, rate limiters, counters, session stores.

## Shared State: Arc, Mutex, and RwLock

When shared state is necessary, Rust provides safe concurrency primitives.

### Arc: Shared Ownership Across Threads

`Arc<T>` (Atomic Reference Counted) lets multiple threads hold a reference to the same data. The data is dropped when the last reference goes away.

```text
config ← SHARED_REF(AppConfig.LOAD())

// Clone the shared reference (cheap -- just increments a counter), not the data
config_clone ← CLONE_REF(config)
SPAWN ASYNC TASK
    // This task owns a reference to the same config
    PRINT "Port: " + config_clone.port
```

`Arc` alone only allows shared immutable access. For mutation, combine it with a lock.

### Mutex: Exclusive Access

`Mutex<T>` ensures only one thread accesses the data at a time. All others wait.

```text
counter ← SHARED_REF(MUTEX(0))

// In a request handler
ACQUIRE LOCK ON counter
    counter.value ← counter.value + 1
// Lock is released automatically
```

**Tokio-aware Mutex:** For async code, use `tokio::sync::Mutex` if you need to hold the lock across `.await` points. For short critical sections (no `.await` inside), `std::sync::Mutex` is faster.

```text
state ← SHARED_REF(ASYNC_MUTEX(NEW AppState))

ASYNC FUNCTION HANDLER(state: Shared<AsyncMutex<AppState>>)
    AWAIT ACQUIRE LOCK ON state
        state.counter ← state.counter + 1
        // Safe to await here because it is an async-aware mutex
        AWAIT state.SAVE_TO_DB()
    RELEASE LOCK
```

### RwLock: Multiple Readers, One Writer

When reads vastly outnumber writes, `RwLock` gives better throughput. Multiple readers can access data simultaneously; writers get exclusive access.

```text
RECORD RateLimiter
    requests: Shared<ReadWriteLock<Map<IpAddr, List<Instant>>>>
    max_requests: Integer
    window: Duration

FUNCTION IS_ALLOWED(limiter: RateLimiter, ip: IpAddr) → Boolean
    ACQUIRE WRITE LOCK ON limiter.requests
    now ← CURRENT_INSTANT()
    window_start ← now - limiter.window

    entry ← requests.GET_OR_INSERT(ip, EMPTY_LIST)
    REMOVE all t FROM entry WHERE t ≤ window_start

    IF LENGTH(entry) < limiter.max_requests THEN
        APPEND now TO entry
        RETURN TRUE
    ELSE
        RETURN FALSE
    RELEASE LOCK

FUNCTION CURRENT_COUNT(limiter: RateLimiter, ip: IpAddr) → Integer
    ACQUIRE READ LOCK ON limiter.requests    // Multiple readers OK
    RETURN LENGTH(requests.GET(ip)) OR 0
    RELEASE LOCK
```

### Avoiding Common Lock Pitfalls

**Hold locks for the minimum time:**

```text
// Bad -- holds lock while doing I/O
ACQUIRE LOCK ON state
    data ← state.GET_DATA()
    result ← EXPENSIVE_COMPUTATION(data)     // Lock held during computation
    state.UPDATE(result)
RELEASE LOCK

// Good -- copy data out, release lock, compute, reacquire
ACQUIRE LOCK ON state
    data ← COPY(state.GET_DATA())
RELEASE LOCK                                  // Release lock
result ← EXPENSIVE_COMPUTATION(data)          // No lock held
ACQUIRE LOCK ON state
    state.UPDATE(result)
RELEASE LOCK
```

**Avoid nested locks** (risk of deadlock): never acquire lock A then lock B if another thread might acquire B then A.

## Message Passing With Channels

When possible, use channels instead of shared state. Each component owns its data and communicates via messages. This is Rust's preferred concurrency model, inspired by Go's "share memory by communicating" philosophy.

### MPSC Channels (Multiple Producer, Single Consumer)

```text
ENUM Command
    IncrementCounter { name: String, amount: Integer }
    GetCounter { name: String, reply: Channel<Integer> }

ASYNC FUNCTION COUNTER_ACTOR(rx: Receiver<Command>)
    counters ← EMPTY Map<String, Integer>

    WHILE cmd ← AWAIT rx.RECEIVE()
        MATCH cmd
            CASE IncrementCounter { name, amount }:
                IF name NOT IN counters THEN
                    counters[name] ← 0
                counters[name] ← counters[name] + amount
            CASE GetCounter { name, reply }:
                value ← counters.GET(name) OR 0
                SEND value TO reply

// Usage from request handlers
ASYNC FUNCTION HANDLE_INCREMENT(tx: Sender<Command>)
    AWAIT tx.SEND(IncrementCounter { name ← "page_views", amount ← 1 })

ASYNC FUNCTION HANDLE_GET_COUNT(tx: Sender<Command>) → Integer
    reply_tx, reply_rx ← NEW_ONESHOT_CHANNEL()
    AWAIT tx.SEND(GetCounter { name ← "page_views", reply ← reply_tx })
    RETURN AWAIT reply_rx.RECEIVE()
```

**Why this works:** The `HashMap` is owned by a single task (the actor). No locks needed. The channel serializes access naturally. Request handlers never touch the data directly.

### Broadcast Channels

For one-to-many communication (e.g., config updates, shutdown signals):

```text
tx ← NEW_BROADCAST_CHANNEL(capacity ← 16)

// Each subscriber gets its own receiver
rx1 ← tx.SUBSCRIBE()
rx2 ← tx.SUBSCRIBE()

// Publisher
tx.SEND(ConfigUpdate.NewRateLimit(100))

// Both rx1 and rx2 receive the update
```

## The Actor Pattern

The actor pattern formalizes message passing: each actor is a task that owns its state, receives messages through a channel, and communicates with other actors only via messages.

```text
RECORD CacheActor
    data: Map<String, CachedItem>
    rx: Receiver<CacheCommand>

ENUM CacheCommand
    Get { key: String, reply: Channel<Optional<CachedItem>> }
    Set { key: String, value: CachedItem }
    Invalidate { key: String }
    Clear

FUNCTION NEW_CACHE_ACTOR(rx: Receiver<CacheCommand>) → CacheActor
    RETURN CacheActor { data ← EMPTY_MAP, rx ← rx }

ASYNC FUNCTION RUN(actor: CacheActor)
    WHILE cmd ← AWAIT actor.rx.RECEIVE()
        MATCH cmd
            CASE Get { key, reply }:
                item ← actor.data.GET(key)
                IF item IS NOT NONE AND NOT item.IS_EXPIRED() THEN
                    SEND COPY(item) TO reply
                ELSE
                    SEND NONE TO reply
            CASE Set { key, value }:
                actor.data[key] ← value
            CASE Invalidate { key }:
                REMOVE key FROM actor.data
            CASE Clear:
                CLEAR actor.data

// Spawn the actor
tx, rx ← NEW_CHANNEL(capacity ← 256)
SPAWN ASYNC TASK RUN(NEW_CACHE_ACTOR(rx))
// Pass tx to handlers that need cache access
```

## Choosing the Right Strategy

| Scenario | Strategy |
|----------|----------|
| Independent request handlers, no shared state | Shared-nothing (default) |
| Read-heavy shared config or cache | `Arc<RwLock<T>>` |
| Write-heavy shared counter or state | Actor with channels |
| Complex state with multiple operations | Actor pattern |
| Short-lived critical sections, no async | `Arc<Mutex<T>>` with `std::sync::Mutex` |
| Lock held across `.await` points | `tokio::sync::Mutex` |
| Fan-out notifications | `broadcast` channel |
| Request-response to a background task | `mpsc` + `oneshot` reply |

## Key Principle

Prefer message passing over shared mutable state. Actors and channels are easier to reason about, harder to deadlock, and scale better. Use `Arc<Mutex<T>>` or `Arc<RwLock<T>>` when the overhead of channels is not justified (simple counters, rarely-changing config).
