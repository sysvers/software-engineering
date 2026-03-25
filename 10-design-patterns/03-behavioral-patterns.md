# Behavioral Patterns

Behavioral patterns deal with communication between objects, distribution of responsibility, and how algorithms are selected and composed at runtime. They answer: "Given that I have these objects, how do they talk to each other and divide work?" In Rust, behavioral patterns take advantage of traits, closures, enums, and iterators -- often more concisely than in OOP languages.

---

## Observer Pattern

### Problem It Solves

When one object changes state, multiple other objects need to react -- without the source knowing about its listeners. A direct function call creates tight coupling: the order system should not import the email module, the inventory module, and the analytics module. The observer decouples the event source from its consumers.

### Rust Implementation

```text
STRUCTURE EventBus<E>
    listeners: map of (event_type: string -> list of callbacks)

    FUNCTION NEW() -> EventBus
        RETURN EventBus { listeners: empty map }

    FUNCTION SUBSCRIBE(event_type: string, callback: function(E))
        ADD callback TO self.listeners[event_type]

    FUNCTION PUBLISH(event_type: string, event: E)
        IF self.listeners HAS event_type
            FOR EACH listener IN self.listeners[event_type]
                CALL listener(event)

// Domain events
STRUCTURE OrderEvent
    order_id: unsigned integer, customer_email: string, total: float, items: list of string

// Usage -- subscribers know nothing about each other
FUNCTION MAIN()
    bus <- NEW EventBus<OrderEvent>()

    // Inventory system
    bus.SUBSCRIBE("order.placed", event =>
        PRINT "Inventory: reserving stock for order " + event.order_id
        FOR item IN event.items: PRINT "  Reserving: " + item)

    // Email system
    bus.SUBSCRIBE("order.placed", event =>
        PRINT "Email: sending confirmation to " + event.customer_email)

    // Analytics
    bus.SUBSCRIBE("order.placed", event =>
        PRINT "Analytics: recording $" + event.total + " order")

    // Fraud detection
    bus.SUBSCRIBE("order.placed", event =>
        IF event.total > 1000.0
            PRINT "Fraud: flagging high-value order " + event.order_id + " for review")

    // Publishing notifies all subscribers
    bus.PUBLISH("order.placed", OrderEvent {
        order_id: 42, customer_email: "alice@example.com",
        total: 1500.0, items: ["Widget A", "Gadget B"] })
```

### Channel-Based Observer (Async/Concurrent)

For multi-threaded or async contexts, use channels instead of callbacks:

```text
ENUMERATION AppEvent
    OrderPlaced { order_id: unsigned integer }
    PaymentReceived { order_id: unsigned integer, amount: float }
    OrderShipped { order_id: unsigned integer, tracking: string }

ASYNC FUNCTION RUN_EVENT_SYSTEM()
    (tx, _) <- NEW broadcast channel<AppEvent>(capacity: 100)

    // Each subscriber gets its own receiver
    inventory_rx <- tx.SUBSCRIBE()
    email_rx <- tx.SUBSCRIBE()

    // Inventory listener
    SPAWN ASYNC TASK
        WHILE event <- inventory_rx.RECV()
            IF event IS AppEvent::OrderPlaced { order_id }
                PRINT "Inventory: processing order " + order_id

    // Email listener
    SPAWN ASYNC TASK
        WHILE event <- email_rx.RECV()
            MATCH event
                CASE OrderPlaced { order_id }: PRINT "Email: order " + order_id + " confirmation sent"
                CASE OrderShipped { order_id, tracking }: PRINT "Email: shipping for " + order_id
                DEFAULT: // ignore

    // Publish events
    tx.SEND(AppEvent::OrderPlaced { order_id: 1 })
    tx.SEND(AppEvent::OrderShipped { order_id: 1, tracking: "1Z999AA10123456784" })
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

```text
INTERFACE CompressionStrategy
    FUNCTION COMPRESS(data: bytes) -> bytes
    FUNCTION DECOMPRESS(data: bytes) -> bytes or DecompressError
    FUNCTION NAME() -> string

STRUCTURE GzipStrategy
    FUNCTION COMPRESS(data) -> GZIP_ENCODE(data)
    FUNCTION DECOMPRESS(data) -> GZIP_DECODE(data)
    FUNCTION NAME() -> "gzip"

STRUCTURE ZstdStrategy { level: integer }
    FUNCTION COMPRESS(data) -> ZSTD_ENCODE(data, self.level)
    FUNCTION DECOMPRESS(data) -> ZSTD_DECODE(data)
    FUNCTION NAME() -> "zstd"

STRUCTURE NoCompression
    FUNCTION COMPRESS(data) -> COPY(data)
    FUNCTION DECOMPRESS(data) -> COPY(data)
    FUNCTION NAME() -> "none"

// The file store does not know which compression is used
STRUCTURE FileStore
    compression: CompressionStrategy, base_path: path

    FUNCTION NEW(path, compression) -> FileStore
        PRINT "FileStore using " + compression.NAME() + " compression"
        RETURN FileStore { compression, base_path: path }

    FUNCTION SAVE(name: string, data: bytes) -> void or IoError
        compressed <- self.compression.COMPRESS(data)
        WRITE compressed TO self.base_path / name

    FUNCTION LOAD(name: string) -> bytes or IoError
        compressed <- READ self.base_path / name
        RETURN self.compression.DECOMPRESS(compressed)

// Choose strategy based on configuration
strategy <- MATCH config.compression
    CASE "gzip": NEW GzipStrategy
    CASE "zstd":  NEW ZstdStrategy { level: 3 }
    CASE "none":  NEW NoCompression
    DEFAULT:      NEW GzipStrategy  // sensible default

store <- FileStore.NEW("/data/files", strategy)
```

### Closure-Based Strategy (Lightweight Alternative)

When the strategy is a single function, closures work well:

```text
TYPE PricingFn <- function(price: float, qty: unsigned integer) -> float

FUNCTION REGULAR_PRICING() -> PricingFn
    RETURN (price, qty) => price * qty

FUNCTION BULK_PRICING(threshold: unsigned integer, discount: float) -> PricingFn
    RETURN (price, qty) =>
        total <- price * qty
        IF qty ≥ threshold THEN total * (1.0 - discount) ELSE total

FUNCTION CALCULATE_ORDER(items: list of (float, unsigned integer), pricing: PricingFn) -> float
    RETURN SUM OF pricing(p, q) FOR EACH (p, q) IN items
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

```text
INTERFACE Command
    FUNCTION EXECUTE() -> void or CommandError
    FUNCTION UNDO() -> void or CommandError
    FUNCTION DESCRIPTION() -> string

// Text editor commands
STRUCTURE InsertText { document: shared locked string, position: size, text: string }
    FUNCTION EXECUTE() -> void or CommandError
        LOCK document
        IF self.position > document.length THEN RETURN Err(InvalidPosition)
        INSERT self.text INTO document AT self.position
        RETURN Ok

    FUNCTION UNDO() -> void or CommandError
        LOCK document
        REMOVE characters from position TO position + text.length IN document
        RETURN Ok

STRUCTURE DeleteText { document: shared locked string, position: size, length: size, deleted: optional string }
    FUNCTION EXECUTE() -> void or CommandError
        LOCK document
        self.deleted <- document[position..position+length]  // save for undo
        REMOVE characters from position TO position + length IN document
        RETURN Ok

    FUNCTION UNDO() -> void or CommandError
        IF self.deleted EXISTS
            LOCK document; INSERT self.deleted AT self.position; RETURN Ok
        ELSE
            RETURN Err(NothingToUndo)

// Command history for undo/redo
STRUCTURE CommandHistory
    executed: list of Command, undone: list of Command

    FUNCTION EXECUTE(cmd: Command) -> void or CommandError
        cmd.EXECUTE()?
        APPEND cmd TO self.executed
        CLEAR self.undone  // new action invalidates redo stack

    FUNCTION UNDO() -> void or CommandError
        IF self.executed IS NOT EMPTY
            cmd <- POP from self.executed
            cmd.UNDO()?
            APPEND cmd TO self.undone
        ELSE RETURN Err(NothingToUndo)

    FUNCTION REDO() -> void or CommandError
        IF self.undone IS NOT EMPTY
            cmd <- POP from self.undone
            cmd.EXECUTE()?
            APPEND cmd TO self.executed
        ELSE RETURN Err(NothingToRedo)
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

```text
ENUMERATION OrderState
    Pending
    Confirmed { confirmed_at: DateTime }
    Processing { started_at: DateTime, worker_id: string }
    Shipped { tracking_number: string, shipped_at: DateTime }
    Delivered { delivered_at: DateTime }
    Cancelled { reason: string, cancelled_at: DateTime }

    FUNCTION CONFIRM(self) -> OrderState or OrderError
        MATCH self
            CASE Pending: RETURN Ok(Confirmed { confirmed_at: NOW() })
            DEFAULT: RETURN Err("Cannot confirm order in state: " + self)

    FUNCTION START_PROCESSING(self, worker: string) -> OrderState or OrderError
        MATCH self
            CASE Confirmed: RETURN Ok(Processing { started_at: NOW(), worker_id: worker })
            DEFAULT: RETURN Err("Cannot process order in state: " + self)

    FUNCTION SHIP(self, tracking: string) -> OrderState or OrderError
        MATCH self
            CASE Processing: RETURN Ok(Shipped { tracking_number: tracking, shipped_at: NOW() })
            DEFAULT: RETURN Err("Cannot ship order in state: " + self)

    FUNCTION DELIVER(self) -> OrderState or OrderError
        MATCH self
            CASE Shipped: RETURN Ok(Delivered { delivered_at: NOW() })
            DEFAULT: RETURN Err("Cannot deliver order in state: " + self)

    FUNCTION CANCEL(self, reason: string) -> OrderState or OrderError
        MATCH self
            CASE Pending OR Confirmed:
                RETURN Ok(Cancelled { reason, cancelled_at: NOW() })
            CASE Processing:
                // Can cancel during processing, but with a fee
                RETURN Ok(Cancelled { reason: reason + " (cancellation fee applied)", cancelled_at: NOW() })
            DEFAULT: RETURN Err("Cannot cancel order in state: " + self)

    FUNCTION IS_TERMINAL(self) -> boolean
        RETURN self IS Delivered OR self IS Cancelled

// Usage
order <- OrderState::Pending
order <- order.CONFIRM()?
order <- order.START_PROCESSING("worker-1")?
order <- order.SHIP("1Z999AA10123456784")?
order <- order.DELIVER()?

// Invalid transitions are caught
pending <- OrderState::Pending
ASSERT pending.SHIP("tracking") IS error  // Cannot ship a pending order
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

```text
// Custom collection with a custom iterator
STRUCTURE Fibonacci
    a: unsigned integer, b: unsigned integer, limit: unsigned integer

    FUNCTION NEW(limit) -> Fibonacci
        RETURN Fibonacci { a: 0, b: 1, limit }

// Implement Iterator for Fibonacci
    FUNCTION NEXT() -> optional unsigned integer
        IF self.a > self.limit
            RETURN None
        current <- self.a
        next <- self.a + self.b
        self.a <- self.b
        self.b <- next
        RETURN Some(current)

// Usage -- works with all iterator adapters
sum <- Fibonacci.NEW(1000000)
    .FILTER(n => n MOD 2 = 0)
    .SUM()

first_ten <- Fibonacci.NEW(MAX_U64)
    .TAKE(10)
    .COLLECT()
```

### A More Practical Example: Paginated API Iterator

```text
STRUCTURE PaginatedApiIterator
    client: ApiClient, endpoint: string, page: unsigned integer
    buffer: list of Record, done: boolean

    FUNCTION NEW(client, endpoint) -> PaginatedApiIterator
        RETURN PaginatedApiIterator { client, endpoint, page: 1, buffer: [], done: false }

// Implement Iterator for PaginatedApiIterator
    FUNCTION NEXT() -> optional (Record or ApiError)
        // Return buffered items first
        IF self.buffer IS NOT EMPTY
            RETURN Some(Ok(POP from self.buffer))

        IF self.done
            RETURN None

        // Fetch next page
        MATCH self.client.GET(self.endpoint, self.page)
            CASE Ok(response):
                IF response.records IS EMPTY
                    self.done <- true; RETURN None
                ELSE
                    self.page <- self.page + 1
                    self.buffer <- response.records
                    REVERSE self.buffer  // so pop gives items in order
                    RETURN Some(Ok(POP from self.buffer))
            CASE Err(e):
                self.done <- true
                RETURN Some(Err(e))

// Callers see a simple iterator, unaware of pagination
all_users <- PaginatedApiIterator.NEW(client, "/api/users")
    .FILTER_MAP(r => r if Ok)
    .TAKE(100)
    .COLLECT()
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
