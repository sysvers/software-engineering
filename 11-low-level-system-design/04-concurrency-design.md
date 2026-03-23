# Concurrency Design

Designing for concurrent access is critical in any system handling multiple requests simultaneously. Rust's ownership system prevents data races at compile time, but you still need to choose the right concurrency strategy for your use case.

## Shared-Nothing Architecture

The simplest and most scalable approach: each request handler gets its own copy of the data it needs. No shared mutable state.

```rust
// Each request gets its own connection from the pool
async fn handle_request(pool: &PgPool) -> Result<Response, Error> {
    let conn = pool.acquire().await?;  // Gets a connection, doesn't share it
    let data = query(&conn).await?;
    Ok(Response::json(data))
}
```

The connection pool itself is shared, but each connection is exclusively owned by one handler. No locking, no contention, no data races.

**When shared-nothing works:** Most web request handlers. Each request reads from the database, computes a response, and returns. No state persists between requests. This should be your default approach.

**When it breaks down:** When you need in-memory state that multiple requests read or write: caches, rate limiters, counters, session stores.

## Shared State: Arc, Mutex, and RwLock

When shared state is necessary, Rust provides safe concurrency primitives.

### Arc: Shared Ownership Across Threads

`Arc<T>` (Atomic Reference Counted) lets multiple threads hold a reference to the same data. The data is dropped when the last reference goes away.

```rust
use std::sync::Arc;

let config = Arc::new(AppConfig::load());

// Clone the Arc (cheap — just increments a counter), not the data
let config_clone = config.clone();
tokio::spawn(async move {
    // This task owns a reference to the same config
    println!("Port: {}", config_clone.port);
});
```

`Arc` alone only allows shared immutable access. For mutation, combine it with a lock.

### Mutex: Exclusive Access

`Mutex<T>` ensures only one thread accesses the data at a time. All others wait.

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0u64));

// In a request handler
let mut guard = counter.lock().unwrap();
*guard += 1;
// Lock is released when `guard` is dropped
```

**Tokio-aware Mutex:** For async code, use `tokio::sync::Mutex` if you need to hold the lock across `.await` points. For short critical sections (no `.await` inside), `std::sync::Mutex` is faster.

```rust
use tokio::sync::Mutex;

let state = Arc::new(Mutex::new(AppState::default()));

async fn handler(state: Arc<Mutex<AppState>>) {
    let mut guard = state.lock().await;
    guard.counter += 1;
    // Safe to .await here because it's a tokio Mutex
    guard.save_to_db().await;
}
```

### RwLock: Multiple Readers, One Writer

When reads vastly outnumber writes, `RwLock` gives better throughput. Multiple readers can access data simultaneously; writers get exclusive access.

```rust
use std::sync::{Arc, RwLock};

struct RateLimiter {
    requests: Arc<RwLock<HashMap<IpAddr, Vec<Instant>>>>,
    max_requests: usize,
    window: Duration,
}

impl RateLimiter {
    fn is_allowed(&self, ip: IpAddr) -> bool {
        let mut requests = self.requests.write().unwrap();
        let now = Instant::now();
        let window_start = now - self.window;

        let entry = requests.entry(ip).or_insert_with(Vec::new);
        entry.retain(|&t| t > window_start);

        if entry.len() < self.max_requests {
            entry.push(now);
            true
        } else {
            false
        }
    }

    fn current_count(&self, ip: IpAddr) -> usize {
        let requests = self.requests.read().unwrap();  // Multiple readers OK
        requests.get(&ip).map(|v| v.len()).unwrap_or(0)
    }
}
```

### Avoiding Common Lock Pitfalls

**Hold locks for the minimum time:**

```rust
// Bad — holds lock while doing I/O
let mut guard = state.lock().unwrap();
let data = guard.get_data();
let result = expensive_computation(data);     // Lock held during computation
guard.update(result);

// Good — copy data out, release lock, compute, reacquire
let data = {
    let guard = state.lock().unwrap();
    guard.get_data().clone()  // Clone and release lock
};
let result = expensive_computation(data);     // No lock held
let mut guard = state.lock().unwrap();
guard.update(result);
```

**Avoid nested locks** (risk of deadlock): never acquire lock A then lock B if another thread might acquire B then A.

## Message Passing With Channels

When possible, use channels instead of shared state. Each component owns its data and communicates via messages. This is Rust's preferred concurrency model, inspired by Go's "share memory by communicating" philosophy.

### MPSC Channels (Multiple Producer, Single Consumer)

```rust
use tokio::sync::{mpsc, oneshot};

enum Command {
    IncrementCounter { name: String, amount: u64 },
    GetCounter { name: String, reply: oneshot::Sender<u64> },
}

async fn counter_actor(mut rx: mpsc::Receiver<Command>) {
    let mut counters: HashMap<String, u64> = HashMap::new();

    while let Some(cmd) = rx.recv().await {
        match cmd {
            Command::IncrementCounter { name, amount } => {
                *counters.entry(name).or_insert(0) += amount;
            }
            Command::GetCounter { name, reply } => {
                let value = counters.get(&name).copied().unwrap_or(0);
                let _ = reply.send(value);
            }
        }
    }
}

// Usage from request handlers
async fn handle_increment(tx: mpsc::Sender<Command>) {
    tx.send(Command::IncrementCounter {
        name: "page_views".into(),
        amount: 1,
    }).await.unwrap();
}

async fn handle_get_count(tx: mpsc::Sender<Command>) -> u64 {
    let (reply_tx, reply_rx) = oneshot::channel();
    tx.send(Command::GetCounter {
        name: "page_views".into(),
        reply: reply_tx,
    }).await.unwrap();
    reply_rx.await.unwrap()
}
```

**Why this works:** The `HashMap` is owned by a single task (the actor). No locks needed. The channel serializes access naturally. Request handlers never touch the data directly.

### Broadcast Channels

For one-to-many communication (e.g., config updates, shutdown signals):

```rust
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel::<ConfigUpdate>(16);

// Each subscriber gets its own receiver
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

// Publisher
tx.send(ConfigUpdate::NewRateLimit(100)).unwrap();

// Both rx1 and rx2 receive the update
```

## The Actor Pattern

The actor pattern formalizes message passing: each actor is a task that owns its state, receives messages through a channel, and communicates with other actors only via messages.

```rust
struct CacheActor {
    data: HashMap<String, CachedItem>,
    rx: mpsc::Receiver<CacheCommand>,
}

enum CacheCommand {
    Get { key: String, reply: oneshot::Sender<Option<CachedItem>> },
    Set { key: String, value: CachedItem },
    Invalidate { key: String },
    Clear,
}

impl CacheActor {
    fn new(rx: mpsc::Receiver<CacheCommand>) -> Self {
        Self { data: HashMap::new(), rx }
    }

    async fn run(mut self) {
        while let Some(cmd) = self.rx.recv().await {
            match cmd {
                CacheCommand::Get { key, reply } => {
                    let item = self.data.get(&key)
                        .filter(|item| !item.is_expired())
                        .cloned();
                    let _ = reply.send(item);
                }
                CacheCommand::Set { key, value } => {
                    self.data.insert(key, value);
                }
                CacheCommand::Invalidate { key } => {
                    self.data.remove(&key);
                }
                CacheCommand::Clear => {
                    self.data.clear();
                }
            }
        }
    }
}

// Spawn the actor
let (tx, rx) = mpsc::channel(256);
tokio::spawn(CacheActor::new(rx).run());
// Pass `tx` to handlers that need cache access
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
