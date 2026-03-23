# Behavioral Patterns

Behavioral patterns deal with communication between objects, distribution of responsibility, and how algorithms are selected and composed at runtime. They answer: "Given that I have these objects, how do they talk to each other and divide work?" In Rust, behavioral patterns take advantage of traits, closures, enums, and iterators -- often more concisely than in OOP languages.

---

## Observer Pattern

### Problem It Solves

When one object changes state, multiple other objects need to react -- without the source knowing about its listeners. A direct function call creates tight coupling: the order system should not import the email module, the inventory module, and the analytics module. The observer decouples the event source from its consumers.

### Rust Implementation

```rust
use std::collections::HashMap;

type Callback<E> = Box<dyn Fn(&E) + Send + Sync>;

pub struct EventBus<E> {
    listeners: HashMap<String, Vec<Callback<E>>>,
}

impl<E> EventBus<E> {
    pub fn new() -> Self {
        Self {
            listeners: HashMap::new(),
        }
    }

    pub fn subscribe(&mut self, event_type: &str, callback: Callback<E>) {
        self.listeners
            .entry(event_type.to_string())
            .or_default()
            .push(callback);
    }

    pub fn publish(&self, event_type: &str, event: &E) {
        if let Some(listeners) = self.listeners.get(event_type) {
            for listener in listeners {
                listener(event);
            }
        }
    }
}

// Domain events
#[derive(Debug)]
struct OrderEvent {
    order_id: u64,
    customer_email: String,
    total: f64,
    items: Vec<String>,
}

// Usage -- subscribers know nothing about each other
fn main() {
    let mut bus: EventBus<OrderEvent> = EventBus::new();

    // Inventory system
    bus.subscribe("order.placed", Box::new(|event| {
        println!("Inventory: reserving stock for order {}", event.order_id);
        for item in &event.items {
            println!("  Reserving: {item}");
        }
    }));

    // Email system
    bus.subscribe("order.placed", Box::new(|event| {
        println!("Email: sending confirmation to {}", event.customer_email);
    }));

    // Analytics
    bus.subscribe("order.placed", Box::new(|event| {
        println!("Analytics: recording ${:.2} order", event.total);
    }));

    // Fraud detection
    bus.subscribe("order.placed", Box::new(|event| {
        if event.total > 1000.0 {
            println!("Fraud: flagging high-value order {} for review", event.order_id);
        }
    }));

    // Publishing notifies all subscribers
    bus.publish("order.placed", &OrderEvent {
        order_id: 42,
        customer_email: "alice@example.com".to_string(),
        total: 1500.0,
        items: vec!["Widget A".into(), "Gadget B".into()],
    });
}
```

### Channel-Based Observer (Async/Concurrent)

For multi-threaded or async contexts, use channels instead of callbacks:

```rust
use tokio::sync::broadcast;

#[derive(Clone, Debug)]
enum AppEvent {
    OrderPlaced { order_id: u64 },
    PaymentReceived { order_id: u64, amount: f64 },
    OrderShipped { order_id: u64, tracking: String },
}

async fn run_event_system() {
    let (tx, _) = broadcast::channel::<AppEvent>(100);

    // Each subscriber gets its own receiver
    let mut inventory_rx = tx.subscribe();
    let mut email_rx = tx.subscribe();

    // Inventory listener
    tokio::spawn(async move {
        while let Ok(event) = inventory_rx.recv().await {
            if let AppEvent::OrderPlaced { order_id } = event {
                println!("Inventory: processing order {order_id}");
            }
        }
    });

    // Email listener
    tokio::spawn(async move {
        while let Ok(event) = email_rx.recv().await {
            match event {
                AppEvent::OrderPlaced { order_id } => {
                    println!("Email: order {order_id} confirmation sent");
                }
                AppEvent::OrderShipped { order_id, tracking } => {
                    println!("Email: shipping notification for {order_id}, tracking: {tracking}");
                }
                _ => {}
            }
        }
    });

    // Publish events
    tx.send(AppEvent::OrderPlaced { order_id: 1 }).unwrap();
    tx.send(AppEvent::OrderShipped {
        order_id: 1,
        tracking: "1Z999AA10123456784".to_string(),
    }).unwrap();
}
```

### Real-World Use Cases

- **Linux kernel notifier chains** -- When a network interface goes up or down, dozens of subsystems (routing, firewall, DHCP) are notified through callback chains. No subsystem knows about the others.
- **tokio::sync::broadcast** -- Tokio provides a multi-producer, multi-consumer broadcast channel used exactly for the observer pattern in async Rust.
- **GUI frameworks (Dioxus, egui)** -- UI components subscribe to state changes and re-render when the state they depend on is modified.
- **Distributed systems (Kafka, RabbitMQ)** -- Message queues are the observer pattern at system scale. Producers publish events; consumers subscribe to topics.

### Pitfalls

1. **Memory leaks from forgotten subscriptions** -- If a subscriber is dropped but its callback remains registered, the callback holds references to dropped data. Use weak references or explicit unsubscribe.
2. **Ordering assumptions** -- Subscribers should not depend on notification order. If subscriber A must run before subscriber B, that is a dependency, not an observation.
3. **Error handling** -- If one subscriber panics or returns an error, should the others still be notified? Define this policy explicitly.
4. **Event storms** -- A subscriber that publishes events in response to events can create infinite loops. Guard against re-entrant publishing.
5. **Single subscriber** -- If there is only one listener, a direct function call is simpler. The observer adds indirection for one-to-many, not one-to-one.

### When to Use / When to Avoid

**Use when:**
- Multiple independent systems need to react to the same event
- The event source should not know about its consumers
- New consumers may be added without modifying the source

**Avoid when:**
- There is only one consumer -- just call it directly
- Consumers need guaranteed ordering -- use a pipeline instead
- Synchronous, transactional behavior is required (observer is fire-and-forget by nature)

---

## Strategy Pattern

### Problem It Solves

You need to swap algorithms at runtime without changing the code that uses them. Different customers get different pricing. Different file formats need different parsers. Different environments need different logging. The algorithm varies, but the calling code stays the same.

### Rust Implementation

```rust
trait CompressionStrategy {
    fn compress(&self, data: &[u8]) -> Vec<u8>;
    fn decompress(&self, data: &[u8]) -> Result<Vec<u8>, DecompressError>;
    fn name(&self) -> &str;
}

struct GzipStrategy;
impl CompressionStrategy for GzipStrategy {
    fn compress(&self, data: &[u8]) -> Vec<u8> {
        // Use flate2 crate
        let mut encoder = GzEncoder::new(Vec::new(), Compression::default());
        encoder.write_all(data).unwrap();
        encoder.finish().unwrap()
    }
    fn decompress(&self, data: &[u8]) -> Result<Vec<u8>, DecompressError> {
        let mut decoder = GzDecoder::new(data);
        let mut buf = Vec::new();
        decoder.read_to_end(&mut buf)?;
        Ok(buf)
    }
    fn name(&self) -> &str { "gzip" }
}

struct ZstdStrategy { level: i32 }
impl CompressionStrategy for ZstdStrategy {
    fn compress(&self, data: &[u8]) -> Vec<u8> {
        zstd::encode_all(data, self.level).unwrap()
    }
    fn decompress(&self, data: &[u8]) -> Result<Vec<u8>, DecompressError> {
        Ok(zstd::decode_all(data)?)
    }
    fn name(&self) -> &str { "zstd" }
}

struct NoCompression;
impl CompressionStrategy for NoCompression {
    fn compress(&self, data: &[u8]) -> Vec<u8> { data.to_vec() }
    fn decompress(&self, data: &[u8]) -> Result<Vec<u8>, DecompressError> { Ok(data.to_vec()) }
    fn name(&self) -> &str { "none" }
}

// The file store does not know which compression is used
struct FileStore {
    compression: Box<dyn CompressionStrategy>,
    base_path: PathBuf,
}

impl FileStore {
    fn new(path: &str, compression: Box<dyn CompressionStrategy>) -> Self {
        println!("FileStore using {} compression", compression.name());
        Self {
            compression,
            base_path: PathBuf::from(path),
        }
    }

    fn save(&self, name: &str, data: &[u8]) -> Result<(), IoError> {
        let compressed = self.compression.compress(data);
        std::fs::write(self.base_path.join(name), &compressed)
    }

    fn load(&self, name: &str) -> Result<Vec<u8>, IoError> {
        let compressed = std::fs::read(self.base_path.join(name))?;
        self.compression.decompress(&compressed)
            .map_err(|e| IoError::new(ErrorKind::InvalidData, e))
    }
}

// Choose strategy based on configuration
let strategy: Box<dyn CompressionStrategy> = match config.compression.as_str() {
    "gzip" => Box::new(GzipStrategy),
    "zstd" => Box::new(ZstdStrategy { level: 3 }),
    "none" => Box::new(NoCompression),
    _ => Box::new(GzipStrategy), // sensible default
};

let store = FileStore::new("/data/files", strategy);
```

### Closure-Based Strategy (Lightweight Alternative)

When the strategy is a single function, closures work well:

```rust
type PricingFn = Box<dyn Fn(f64, u32) -> f64>;

fn regular_pricing() -> PricingFn {
    Box::new(|price, qty| price * qty as f64)
}

fn bulk_pricing(threshold: u32, discount: f64) -> PricingFn {
    Box::new(move |price, qty| {
        let total = price * qty as f64;
        if qty >= threshold { total * (1.0 - discount) } else { total }
    })
}

fn calculate_order(items: &[(f64, u32)], pricing: &PricingFn) -> f64 {
    items.iter().map(|(p, q)| pricing(*p, *q)).sum()
}
```

### Real-World Use Cases

- **Stripe payment processing** -- Hundreds of payment processors worldwide, each with its own API. A common `PaymentProcessor` interface with a different strategy per processor. Adding a new payment method means adding a new strategy.
- **serde serialization formats** -- `serde` itself uses the strategy pattern. The `Serializer` trait is the strategy interface; `serde_json::Serializer`, `serde_yaml::Serializer`, etc. are concrete strategies.
- **Sorting algorithms** -- Rust's `sort_by` takes a comparison closure, which is the strategy pattern via closures. Different orderings without changing the sort algorithm.
- **Retry policies** -- HTTP clients like `reqwest` with `tower::retry` let you plug in different retry strategies (exponential backoff, fixed delay, jitter).

### Pitfalls

1. **Premature abstraction** -- Do not create a strategy trait until you have at least 2 concrete implementations, ideally 3.
2. **Strategy explosion** -- If every minor variation gets its own strategy, you end up with dozens of tiny structs. Consider parameterizing a single strategy instead.
3. **Context leakage** -- Strategies should be self-contained. If a strategy needs access to half the application's state, it is doing too much.
4. **Enum vs. trait** -- If the set of strategies is fixed and known at compile time, use an enum. Trait objects add heap allocation and dynamic dispatch overhead unnecessarily.

### When to Use / When to Avoid

**Use when:**
- The algorithm must be chosen at runtime
- Multiple algorithms share the same interface
- You want to add new algorithms without modifying existing code

**Avoid when:**
- There is one algorithm and no plan for more
- The "strategies" differ in one parameter -- just parameterize the function
- Compile-time selection is sufficient -- use generics or enums

---

## Command Pattern

### Problem It Solves

You need to encapsulate a request as an object, enabling you to parameterize clients with different requests, queue requests, log them, or support undo/redo. The command decouples "what to do" from "when to do it" and "who does it."

### Rust Implementation

```rust
trait Command: Send {
    fn execute(&mut self) -> Result<(), CommandError>;
    fn undo(&mut self) -> Result<(), CommandError>;
    fn description(&self) -> &str;
}

// Text editor commands
struct InsertText {
    document: Arc<Mutex<String>>,
    position: usize,
    text: String,
}

impl Command for InsertText {
    fn execute(&mut self) -> Result<(), CommandError> {
        let mut doc = self.document.lock().unwrap();
        if self.position > doc.len() {
            return Err(CommandError::InvalidPosition);
        }
        doc.insert_str(self.position, &self.text);
        Ok(())
    }

    fn undo(&mut self) -> Result<(), CommandError> {
        let mut doc = self.document.lock().unwrap();
        doc.replace_range(self.position..self.position + self.text.len(), "");
        Ok(())
    }

    fn description(&self) -> &str { "Insert text" }
}

struct DeleteText {
    document: Arc<Mutex<String>>,
    position: usize,
    length: usize,
    deleted: Option<String>, // saved for undo
}

impl Command for DeleteText {
    fn execute(&mut self) -> Result<(), CommandError> {
        let mut doc = self.document.lock().unwrap();
        self.deleted = Some(doc[self.position..self.position + self.length].to_string());
        doc.replace_range(self.position..self.position + self.length, "");
        Ok(())
    }

    fn undo(&mut self) -> Result<(), CommandError> {
        if let Some(ref deleted) = self.deleted {
            let mut doc = self.document.lock().unwrap();
            doc.insert_str(self.position, deleted);
            Ok(())
        } else {
            Err(CommandError::NothingToUndo)
        }
    }

    fn description(&self) -> &str { "Delete text" }
}

// Command history for undo/redo
struct CommandHistory {
    executed: Vec<Box<dyn Command>>,
    undone: Vec<Box<dyn Command>>,
}

impl CommandHistory {
    fn new() -> Self {
        Self { executed: vec![], undone: vec![] }
    }

    fn execute(&mut self, mut cmd: Box<dyn Command>) -> Result<(), CommandError> {
        cmd.execute()?;
        self.executed.push(cmd);
        self.undone.clear(); // new action invalidates redo stack
        Ok(())
    }

    fn undo(&mut self) -> Result<(), CommandError> {
        if let Some(mut cmd) = self.executed.pop() {
            cmd.undo()?;
            self.undone.push(cmd);
            Ok(())
        } else {
            Err(CommandError::NothingToUndo)
        }
    }

    fn redo(&mut self) -> Result<(), CommandError> {
        if let Some(mut cmd) = self.undone.pop() {
            cmd.execute()?;
            self.executed.push(cmd);
            Ok(())
        } else {
            Err(CommandError::NothingToRedo)
        }
    }
}
```

### Real-World Use Cases

- **Text editors and IDEs** -- Every action (type, delete, paste, format) is a command. This enables undo/redo, macro recording, and collaborative editing.
- **Database migrations** -- Each migration is a command with `up()` and `down()` methods. Tools like `diesel_migrations` and `sqlx` migrations use this pattern.
- **Job queues** -- Background job systems (Sidekiq, Celery) serialize commands and execute them later on worker processes. The command pattern enables queuing, retrying, and auditing.
- **Game engines** -- Player actions are commands that can be replayed (replay system), undone (undo move), or sent over the network (multiplayer synchronization).

### Pitfalls

1. **Command bloat** -- Every trivial action becomes a struct with `execute` and `undo`. For simple cases, closures may suffice.
2. **Undo complexity** -- Not every action is easily reversible. Deleting a file, sending an email, or charging a credit card cannot be trivially undone. Design undo carefully.
3. **State capture** -- Commands must capture enough state to undo themselves. Forgetting to save the deleted text means undo is impossible.
4. **Concurrency** -- If multiple threads execute commands on shared state, the command history must be synchronized.

### When to Use / When to Avoid

**Use when:**
- You need undo/redo functionality
- Actions need to be queued, scheduled, or replayed
- You need an audit log of all operations
- Macro recording (replay a sequence of commands)

**Avoid when:**
- Actions are fire-and-forget with no undo requirement
- The overhead of wrapping every action in a struct is not justified
- Simple function calls suffice

---

## State Pattern

### Problem It Solves

An object behaves differently based on its internal state. The naive approach -- a giant `match` statement in every method -- grows unwieldy as states and transitions multiply. The state pattern encapsulates state-specific behavior and makes transitions explicit.

### Rust Implementation (Enum-Based)

Rust's enums with data are the natural fit for the state pattern:

```rust
use chrono::{DateTime, Utc};

#[derive(Debug)]
enum OrderState {
    Pending,
    Confirmed { confirmed_at: DateTime<Utc> },
    Processing { started_at: DateTime<Utc>, worker_id: String },
    Shipped { tracking_number: String, shipped_at: DateTime<Utc> },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String, cancelled_at: DateTime<Utc> },
}

#[derive(Debug)]
struct OrderError(String);

impl OrderState {
    fn confirm(self) -> Result<OrderState, OrderError> {
        match self {
            OrderState::Pending => Ok(OrderState::Confirmed {
                confirmed_at: Utc::now(),
            }),
            other => Err(OrderError(format!("Cannot confirm order in state: {other:?}"))),
        }
    }

    fn start_processing(self, worker: String) -> Result<OrderState, OrderError> {
        match self {
            OrderState::Confirmed { .. } => Ok(OrderState::Processing {
                started_at: Utc::now(),
                worker_id: worker,
            }),
            other => Err(OrderError(format!("Cannot process order in state: {other:?}"))),
        }
    }

    fn ship(self, tracking: String) -> Result<OrderState, OrderError> {
        match self {
            OrderState::Processing { .. } => Ok(OrderState::Shipped {
                tracking_number: tracking,
                shipped_at: Utc::now(),
            }),
            other => Err(OrderError(format!("Cannot ship order in state: {other:?}"))),
        }
    }

    fn deliver(self) -> Result<OrderState, OrderError> {
        match self {
            OrderState::Shipped { .. } => Ok(OrderState::Delivered {
                delivered_at: Utc::now(),
            }),
            other => Err(OrderError(format!("Cannot deliver order in state: {other:?}"))),
        }
    }

    fn cancel(self, reason: String) -> Result<OrderState, OrderError> {
        match self {
            OrderState::Pending | OrderState::Confirmed { .. } => {
                Ok(OrderState::Cancelled {
                    reason,
                    cancelled_at: Utc::now(),
                })
            }
            OrderState::Processing { .. } => {
                // Can cancel during processing, but with a fee
                Ok(OrderState::Cancelled {
                    reason: format!("{reason} (cancellation fee applied)"),
                    cancelled_at: Utc::now(),
                })
            }
            other => Err(OrderError(format!("Cannot cancel order in state: {other:?}"))),
        }
    }

    fn is_terminal(&self) -> bool {
        matches!(self, OrderState::Delivered { .. } | OrderState::Cancelled { .. })
    }
}

// Usage
let order = OrderState::Pending;
let order = order.confirm()?;
let order = order.start_processing("worker-1".to_string())?;
let order = order.ship("1Z999AA10123456784".to_string())?;
let order = order.deliver()?;

// Invalid transitions are caught
let pending = OrderState::Pending;
assert!(pending.ship("tracking".to_string()).is_err()); // Cannot ship a pending order
```

### Real-World Use Cases

- **HTTP connection lifecycle** -- A connection moves through states: `Idle -> Sending -> AwaitingResponse -> Received -> Idle`. Each state has different valid operations.
- **TCP state machine** -- The TCP protocol itself is a state machine (LISTEN, SYN_SENT, ESTABLISHED, FIN_WAIT, etc.) implemented with the state pattern in every TCP stack.
- **Payment processing** -- Payments go through `Created -> Authorized -> Captured -> Settled` (or `Declined`/`Refunded` branches). Each state allows different operations.
- **CI/CD pipelines** -- Build stages (`Queued -> Running -> Passed/Failed`) with rules about what can happen in each stage.

### Pitfalls

1. **State explosion** -- If you have 20 states with 30 transitions, the match arms become unwieldy. Consider hierarchical states or state charts.
2. **Missing transitions** -- The compiler warns about non-exhaustive matches, which is a strength. But forgetting to add a new state to every method is still easy.
3. **Shared behavior** -- If most states share 90% of their behavior, the state pattern forces duplication. Extract shared behavior into helper methods.
4. **Enum vs. typestate** -- For compile-time enforcement, use the typestate pattern (see 04-rust-specific-patterns.md). Enums catch invalid transitions at runtime; typestates catch them at compile time.

### When to Use / When to Avoid

**Use when:**
- An object has distinct behavioral modes (states)
- Transitions between states follow rules that must be enforced
- State-specific behavior would otherwise be scattered across large match blocks

**Avoid when:**
- There are only 2 states -- a boolean flag is simpler
- The "states" do not actually have different behavior
- Compile-time enforcement is needed -- use the typestate pattern instead

---

## Iterator Pattern

### Problem It Solves

You need to traverse a collection without exposing its internal structure. Different collections (arrays, linked lists, trees, database result sets) should be traversable through a uniform interface. Consumers should not know or care how the data is stored.

### Rust Implementation

Rust has first-class support for iterators through the `Iterator` trait:

```rust
// Custom collection with a custom iterator
struct Fibonacci {
    a: u64,
    b: u64,
    limit: u64,
}

impl Fibonacci {
    fn new(limit: u64) -> Self {
        Self { a: 0, b: 1, limit }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        if self.a > self.limit {
            return None;
        }
        let current = self.a;
        let next = self.a + self.b;
        self.a = self.b;
        self.b = next;
        Some(current)
    }
}

// Usage -- works with all iterator adapters
let sum: u64 = Fibonacci::new(1_000_000)
    .filter(|n| n % 2 == 0)
    .sum();

let first_ten: Vec<u64> = Fibonacci::new(u64::MAX)
    .take(10)
    .collect();
```

### A More Practical Example: Paginated API Iterator

```rust
struct PaginatedApiIterator {
    client: ApiClient,
    endpoint: String,
    page: u32,
    buffer: Vec<Record>,
    done: bool,
}

impl PaginatedApiIterator {
    fn new(client: ApiClient, endpoint: &str) -> Self {
        Self {
            client,
            endpoint: endpoint.to_string(),
            page: 1,
            buffer: Vec::new(),
            done: false,
        }
    }
}

impl Iterator for PaginatedApiIterator {
    type Item = Result<Record, ApiError>;

    fn next(&mut self) -> Option<Self::Item> {
        // Return buffered items first
        if let Some(record) = self.buffer.pop() {
            return Some(Ok(record));
        }

        if self.done {
            return None;
        }

        // Fetch next page
        match self.client.get(&self.endpoint, self.page) {
            Ok(response) => {
                if response.records.is_empty() {
                    self.done = true;
                    None
                } else {
                    self.page += 1;
                    self.buffer = response.records;
                    self.buffer.reverse(); // so pop() gives items in order
                    self.buffer.pop().map(Ok)
                }
            }
            Err(e) => {
                self.done = true;
                Some(Err(e))
            }
        }
    }
}

// Callers see a simple iterator, unaware of pagination
let all_users: Vec<User> = PaginatedApiIterator::new(client, "/api/users")
    .filter_map(|r| r.ok())
    .take(100)
    .collect();
```

### Real-World Use Cases

- **Rust standard library** -- Every collection (`Vec`, `HashMap`, `BTreeSet`, `String`) implements `IntoIterator`. The iterator ecosystem (`.map()`, `.filter()`, `.collect()`, `.fold()`) is one of Rust's most powerful features.
- **diesel query results** -- Database query results implement `Iterator`, letting you process rows lazily without loading the entire result set into memory.
- **walkdir** -- The `walkdir` crate provides an iterator over directory entries, traversing the filesystem lazily.
- **csv::Reader** -- The `csv` crate returns an iterator over rows, parsing each row on demand.
- **rayon** -- `rayon`'s parallel iterators (`par_iter()`) have the same interface as standard iterators but execute in parallel. Switching from sequential to parallel is a one-word change.

### Pitfalls

1. **Lazy evaluation surprises** -- Iterators are lazy in Rust. `vec.iter().map(|x| x * 2)` does nothing until consumed. This trips up newcomers who expect immediate execution.
2. **Iterator invalidation** -- Modifying a collection while iterating over it is a compile-time error in Rust (the borrow checker prevents it). This is a strength, not a pitfall -- but be aware of it.
3. **Infinite iterators** -- Iterators like `std::iter::repeat(42)` never terminate. Always use `.take(n)` or other limiting adapters.
4. **Collect into wrong type** -- `.collect()` uses type inference. `let v: Vec<_> = ...` and `let s: String = ...` produce different results from the same iterator. Be explicit about the target type.

### When to Use / When to Avoid

**Use when:**
- You need to traverse any collection uniformly
- Lazy evaluation is desired (process items on demand)
- You want to chain transformations (map, filter, fold)
- The data source is potentially infinite or very large

**Avoid when:**
- You need random access (use indexing instead)
- The collection is small and a simple `for` loop is clearer
- You need to modify the collection during traversal (use `retain`, `drain`, or build a new collection)
