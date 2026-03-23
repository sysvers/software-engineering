# 10 - Design Patterns

## Concepts

### What are Design Patterns?

Design patterns are reusable solutions to common problems in software design. They're not code you copy-paste — they're templates for solving a category of problem.

The concept was popularized by the "Gang of Four" (GoF) book *Design Patterns: Elements of Reusable Object-Oriented Software* (1994). While the original patterns are OOP-centric, many translate well to other paradigms — including Rust's trait-based system.

**Why patterns matter:** They give teams a shared vocabulary. Saying "use the Builder pattern here" communicates an entire design approach in four words. Without patterns, every design discussion starts from scratch.

**The danger of patterns:** Over-application. Not every problem needs a pattern. Using a pattern where simple code suffices adds unnecessary complexity. Patterns are tools, not goals.

### Creational Patterns

Creational patterns deal with object/struct creation, providing flexibility in *what* gets created, *how*, and *when*.

#### Builder Pattern

**Problem:** A struct has many fields, some optional. Constructors with 10+ parameters are unreadable and error-prone.

**Solution:** A separate builder struct that accumulates configuration and produces the final object.

```rust
pub struct HttpRequest {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout_ms: u64,
}

pub struct HttpRequestBuilder {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout_ms: u64,
}

impl HttpRequestBuilder {
    pub fn new(url: &str) -> Self {
        Self {
            url: url.to_string(),
            method: "GET".to_string(),
            headers: vec![],
            body: None,
            timeout_ms: 30_000,
        }
    }

    pub fn method(mut self, method: &str) -> Self {
        self.method = method.to_string();
        self
    }

    pub fn header(mut self, key: &str, value: &str) -> Self {
        self.headers.push((key.to_string(), value.to_string()));
        self
    }

    pub fn body(mut self, body: &str) -> Self {
        self.body = Some(body.to_string());
        self
    }

    pub fn timeout(mut self, ms: u64) -> Self {
        self.timeout_ms = ms;
        self
    }

    pub fn build(self) -> HttpRequest {
        HttpRequest {
            url: self.url,
            method: self.method,
            headers: self.headers,
            body: self.body,
            timeout_ms: self.timeout_ms,
        }
    }
}

// Usage — reads like a sentence
let request = HttpRequestBuilder::new("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .body(r#"{"name": "Alice"}"#)
    .timeout(5_000)
    .build();
```

**Where you see this in the real world:** `reqwest::Client::builder()`, `tokio::runtime::Builder`, `clap::Command::new()`.

#### Factory Pattern

**Problem:** You need to create objects of different types based on runtime conditions, but callers shouldn't know about the concrete types.

**Solution:** A function or trait that returns a trait object based on input.

```rust
trait NotificationSender: Send + Sync {
    fn send(&self, to: &str, message: &str) -> Result<(), SendError>;
}

struct EmailSender { /* ... */ }
struct SmsSender { /* ... */ }
struct SlackSender { /* ... */ }

impl NotificationSender for EmailSender { /* ... */ }
impl NotificationSender for SmsSender { /* ... */ }
impl NotificationSender for SlackSender { /* ... */ }

fn create_sender(channel: &str) -> Box<dyn NotificationSender> {
    match channel {
        "email" => Box::new(EmailSender::new()),
        "sms" => Box::new(SmsSender::new()),
        "slack" => Box::new(SlackSender::new()),
        _ => panic!("Unknown channel: {channel}"),
    }
}
```

#### Singleton (and Why It's Often an Anti-Pattern)

**Problem:** You need exactly one instance of something (database connection pool, configuration, logger).

**In traditional OOP:** A class with a private constructor and a static `getInstance()` method.

**Why it's problematic:**
- Hidden global state makes testing hard
- Tight coupling — every user depends on the singleton directly
- Concurrency issues (who initializes it? thread safety?)

**In Rust, prefer these alternatives:**
- Pass shared resources explicitly as function parameters
- Use `Arc<T>` for shared ownership across threads
- Use `once_cell::sync::Lazy` or `std::sync::OnceLock` for one-time initialization

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<Config> = OnceLock::new();

fn init_config(path: &str) {
    let config = load_config(path).expect("Failed to load config");
    CONFIG.set(config).expect("Config already initialized");
}

fn get_config() -> &'static Config {
    CONFIG.get().expect("Config not initialized")
}
```

**Better approach — dependency injection:**
```rust
// Instead of accessing a global, pass the config explicitly
fn start_server(config: &Config, pool: &PgPool) -> Result<(), ServerError> {
    // ...
}
```

### Structural Patterns

Structural patterns deal with composing structs and traits to form larger structures.

#### Adapter Pattern

**Problem:** You have an existing interface, but a component expects a different one. You can't modify either.

**Solution:** A wrapper that translates between the two interfaces.

```rust
// External library's interface — you can't change this
struct LegacyPaymentProcessor;
impl LegacyPaymentProcessor {
    fn process_usd_cents(&self, amount: u64, card: &str) -> bool {
        // Legacy implementation...
        true
    }
}

// Your application's interface
trait PaymentGateway {
    fn charge(&self, amount_dollars: f64, card_number: &str) -> Result<(), PaymentError>;
}

// Adapter — bridges the gap
struct LegacyPaymentAdapter {
    processor: LegacyPaymentProcessor,
}

impl PaymentGateway for LegacyPaymentAdapter {
    fn charge(&self, amount_dollars: f64, card_number: &str) -> Result<(), PaymentError> {
        let cents = (amount_dollars * 100.0) as u64;
        if self.processor.process_usd_cents(cents, card_number) {
            Ok(())
        } else {
            Err(PaymentError::Declined)
        }
    }
}
```

**Real-world use:** Wrapping third-party libraries to match your application's interfaces, enabling you to swap implementations without changing callers.

#### Decorator Pattern

**Problem:** You want to add behavior to an object without modifying its code.

**Solution:** A wrapper that implements the same trait, adding behavior before/after delegating to the wrapped object.

```rust
trait Logger: Send + Sync {
    fn log(&self, message: &str);
}

struct ConsoleLogger;
impl Logger for ConsoleLogger {
    fn log(&self, message: &str) {
        println!("{message}");
    }
}

// Decorator: adds timestamps
struct TimestampLogger<L: Logger> {
    inner: L,
}

impl<L: Logger> Logger for TimestampLogger<L> {
    fn log(&self, message: &str) {
        let now = chrono::Utc::now().format("%Y-%m-%d %H:%M:%S");
        self.inner.log(&format!("[{now}] {message}"));
    }
}

// Decorator: adds log levels
struct LevelLogger<L: Logger> {
    inner: L,
    level: String,
}

impl<L: Logger> Logger for LevelLogger<L> {
    fn log(&self, message: &str) {
        self.inner.log(&format!("[{}] {message}", self.level));
    }
}

// Composable — stack decorators
let logger = TimestampLogger {
    inner: LevelLogger {
        inner: ConsoleLogger,
        level: "INFO".to_string(),
    },
};
logger.log("Server started");
// Output: [2024-03-15 10:30:00] [INFO] Server started
```

#### Facade Pattern

**Problem:** A subsystem has a complex API with many components. Most callers only need simple operations.

**Solution:** A simplified interface that hides the complexity.

```rust
// Complex subsystem
struct CpuMonitor { /* ... */ }
struct MemoryMonitor { /* ... */ }
struct DiskMonitor { /* ... */ }
struct NetworkMonitor { /* ... */ }

// Facade — simple interface for common operations
pub struct SystemHealth {
    cpu: CpuMonitor,
    memory: MemoryMonitor,
    disk: DiskMonitor,
    network: NetworkMonitor,
}

impl SystemHealth {
    pub fn is_healthy(&self) -> bool {
        self.cpu.usage() < 90.0
            && self.memory.available_percent() > 10.0
            && self.disk.free_percent() > 5.0
            && self.network.is_reachable()
    }

    pub fn summary(&self) -> HealthReport {
        HealthReport {
            cpu_usage: self.cpu.usage(),
            memory_free: self.memory.available_percent(),
            disk_free: self.disk.free_percent(),
            network_ok: self.network.is_reachable(),
        }
    }
}
```

### Behavioral Patterns

Behavioral patterns deal with communication between objects and distribution of responsibility.

#### Strategy Pattern

**Problem:** You need to swap algorithms at runtime without changing the code that uses them.

**Solution:** Define a family of algorithms behind a trait, and pass the desired implementation.

```rust
trait PricingStrategy {
    fn calculate(&self, base_price: f64, quantity: u32) -> f64;
}

struct RegularPricing;
impl PricingStrategy for RegularPricing {
    fn calculate(&self, base_price: f64, quantity: u32) -> f64 {
        base_price * quantity as f64
    }
}

struct BulkPricing { threshold: u32, discount: f64 }
impl PricingStrategy for BulkPricing {
    fn calculate(&self, base_price: f64, quantity: u32) -> f64 {
        let total = base_price * quantity as f64;
        if quantity >= self.threshold {
            total * (1.0 - self.discount)
        } else {
            total
        }
    }
}

struct SeasonalPricing { multiplier: f64 }
impl PricingStrategy for SeasonalPricing {
    fn calculate(&self, base_price: f64, quantity: u32) -> f64 {
        base_price * self.multiplier * quantity as f64
    }
}

// The order doesn't know which pricing strategy is used
fn calculate_order_total(
    items: &[(f64, u32)],
    strategy: &dyn PricingStrategy,
) -> f64 {
    items.iter()
        .map(|(price, qty)| strategy.calculate(*price, *qty))
        .sum()
}
```

#### Observer Pattern

**Problem:** When one object changes state, multiple other objects need to be notified — without tight coupling.

**Solution:** Maintain a list of listeners and notify them when state changes.

```rust
type Callback = Box<dyn Fn(&OrderEvent) + Send + Sync>;

pub struct EventBus {
    listeners: Vec<Callback>,
}

impl EventBus {
    pub fn new() -> Self {
        Self { listeners: vec![] }
    }

    pub fn subscribe(&mut self, callback: Callback) {
        self.listeners.push(callback);
    }

    pub fn publish(&self, event: &OrderEvent) {
        for listener in &self.listeners {
            listener(event);
        }
    }
}

// Usage
let mut bus = EventBus::new();

bus.subscribe(Box::new(|event| {
    println!("Inventory: updating stock for order {}", event.order_id);
}));

bus.subscribe(Box::new(|event| {
    println!("Email: sending confirmation for order {}", event.order_id);
}));

bus.subscribe(Box::new(|event| {
    println!("Analytics: tracking order {}", event.order_id);
}));

bus.publish(&OrderEvent { order_id: 42 });
```

#### State Pattern

**Problem:** An object behaves differently based on its internal state. The naive approach is a massive `match` statement that grows with every state.

**Solution:** Represent each state as a separate type/struct, with transitions between them.

```rust
// State machine for an order
enum OrderState {
    Pending,
    Confirmed { confirmed_at: DateTime<Utc> },
    Shipped { tracking_number: String },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String },
}

impl OrderState {
    fn confirm(self) -> Result<OrderState, OrderError> {
        match self {
            OrderState::Pending => Ok(OrderState::Confirmed {
                confirmed_at: Utc::now(),
            }),
            _ => Err(OrderError::InvalidTransition("Can only confirm pending orders")),
        }
    }

    fn ship(self, tracking: String) -> Result<OrderState, OrderError> {
        match self {
            OrderState::Confirmed { .. } => Ok(OrderState::Shipped {
                tracking_number: tracking,
            }),
            _ => Err(OrderError::InvalidTransition("Can only ship confirmed orders")),
        }
    }

    fn cancel(self, reason: String) -> Result<OrderState, OrderError> {
        match self {
            OrderState::Pending | OrderState::Confirmed { .. } => {
                Ok(OrderState::Cancelled { reason })
            }
            _ => Err(OrderError::InvalidTransition("Cannot cancel shipped/delivered orders")),
        }
    }
}
```

### Rust-Specific Patterns

#### Newtype Pattern

Wrap a primitive type to give it semantic meaning and type safety.

```rust
// Without newtype — easy to mix up
fn transfer(from: u64, to: u64, amount: u64) { /* ... */ }
transfer(amount, from_id, to_id); // Compiles! But wrong!

// With newtype — compiler catches mistakes
struct UserId(u64);
struct Amount(u64);

fn transfer(from: UserId, to: UserId, amount: Amount) { /* ... */ }
transfer(Amount(100), UserId(1), UserId(2)); // Compiler error!
```

#### Typestate Pattern

Use the type system to enforce valid state transitions at compile time.

```rust
struct Unlocked;
struct Locked;

struct Door<State> {
    _state: std::marker::PhantomData<State>,
}

impl Door<Locked> {
    fn unlock(self, key: &Key) -> Result<Door<Unlocked>, LockError> {
        // verify key...
        Ok(Door { _state: std::marker::PhantomData })
    }
}

impl Door<Unlocked> {
    fn lock(self) -> Door<Locked> {
        Door { _state: std::marker::PhantomData }
    }

    fn open(&self) {
        println!("Door is open");
    }
}

// Compile-time enforcement:
let door: Door<Locked> = Door { _state: std::marker::PhantomData };
// door.open(); // ← Compiler error! Can't open a locked door
let door = door.unlock(&key)?;
door.open(); // ✅ Works — door is unlocked
```

**Real-world use:** HTTP request builders (can't send a request without setting the URL), database transaction builders (can't commit without beginning), protocol state machines.

### Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| **God Object** | One struct that does everything, knows everything | Split into focused, single-responsibility structs |
| **Spaghetti Code** | No clear structure, everything calls everything | Establish layers and dependency direction |
| **Lava Flow** | Dead code and unused abstractions nobody dares delete | Delete it. Git remembers. |
| **Golden Hammer** | Using one pattern for every problem | Choose patterns based on the problem, not familiarity |
| **Premature Abstraction** | Adding patterns before complexity justifies them | Wait for duplication and pain before abstracting |
| **Pattern Mania** | Using 5 patterns where simple functions would suffice | Simple code > clever code |

## Business Value

- **Shared vocabulary**: Patterns give teams a common language for design discussions. "Use Strategy here" communicates instantly, saving hours of whiteboard sessions.
- **Faster onboarding**: Engineers who know common patterns can understand unfamiliar codebases faster. The Builder pattern in any language looks recognizable.
- **Reduced design risk**: Patterns are battle-tested solutions. Using them correctly means you're building on proven approaches, not inventing from scratch.
- **Maintainability**: Patterns like Strategy and Observer enable extending behavior without modifying existing code (Open-Closed Principle), reducing regression risk.

## Real-World Examples

### Tokio's Builder Pattern
Tokio (Rust's async runtime) uses the Builder pattern extensively. `tokio::runtime::Builder` lets you configure thread count, thread names, enable/disable I/O and timer drivers — all through a fluent API. This makes the complex configuration of an async runtime approachable and self-documenting.

### Stripe's Strategy Pattern for Payment Processing
Stripe processes payments through hundreds of different payment processors worldwide. Each processor has its own API, error codes, and retry logic. They use the Strategy pattern: a common `PaymentProcessor` interface with a different implementation for each processor. Adding a new payment method means adding a new strategy, not modifying existing code.

### Linux Kernel's Observer Pattern
The Linux kernel uses the observer pattern (called "notifier chains") extensively. When a network interface goes up or down, dozens of subsystems need to be notified (routing table, firewall, DHCP, etc.). The notifier chain lets subsystems subscribe to events without the network code knowing about any of them.

### AWS SDK's Builder Pattern
Every AWS SDK uses the Builder pattern for API requests. Creating an S3 PutObject request involves setting the bucket, key, body, content type, ACL, metadata, and more. The builder makes this manageable and self-documenting, with IDE autocomplete guiding developers through the options.

## Common Mistakes & Pitfalls

- **Pattern for pattern's sake** — Applying the Observer pattern to notify a single listener. If there's only one subscriber, a direct function call is simpler.

- **Premature abstraction** — Introducing a Strategy pattern when there's only one strategy. Wait until you have 2-3 concrete strategies before abstracting.

- **Overusing inheritance** — GoF patterns were designed for OOP with inheritance. In Rust, prefer composition and traits. Don't force OOP patterns into Rust — use Rust-idiomatic alternatives.

- **Ignoring Rust's type system** — Rust's enums, pattern matching, and ownership system solve many problems that require complex patterns in other languages. A Rust `enum` often replaces the State pattern entirely.

- **Not knowing when to stop** — A system with 15 patterns and 3 lines of actual logic is over-engineered. Patterns should reduce complexity, not add it.

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **Heavy pattern usage** | Extensible, well-structured, familiar | Over-engineered for simple problems, harder to follow |
| **Minimal patterns** | Simple, direct, easy to understand | May need refactoring as complexity grows |
| **Rust-idiomatic (enums, traits)** | Leverages type system, compile-time safety | May be unfamiliar to engineers from OOP backgrounds |
| **GoF patterns verbatim** | Well-documented, universally known | Often awkward in Rust, may fight the borrow checker |

## When to Use / When Not to Use

**Use design patterns when:**
- You recognize a recurring problem that a pattern solves
- The pattern reduces complexity (not adds it)
- Multiple team members need to understand and extend the design
- The code will be maintained for years

**Skip design patterns when:**
- Simple functions and structs solve the problem
- There's only one implementation (no need for Strategy with one strategy)
- The code is a prototype or script
- Adding the pattern makes the code harder to understand

**Rust-specific guidance:**
- Prefer `enum` over State/Strategy when the set of variants is known at compile time
- Prefer traits over abstract base classes
- Use the newtype pattern liberally — it's cheap and prevents bugs
- Typestate pattern is unique to Rust — use it for compile-time state machine enforcement

## Key Takeaways

1. Design patterns are vocabulary, not mandates. Know them, but apply them only when they reduce complexity.
2. Builder is the most commonly used pattern in Rust. Use it for any struct with 4+ fields or optional configuration.
3. Rust's enums + pattern matching often replace the State, Strategy, and Visitor patterns more elegantly than traditional OOP approaches.
4. The newtype pattern is cheap, powerful, and underused. Wrap primitive types to prevent mix-ups.
5. The typestate pattern leverages Rust's type system for compile-time enforcement of valid state transitions — something impossible in most languages.
6. Anti-patterns are just as important to recognize as patterns. The most common: premature abstraction and pattern mania.

## Further Reading

- **Books:**
  - *Design Patterns: Elements of Reusable Object-Oriented Software* — Gang of Four (1994) — The original patterns book
  - *Rust Design Patterns* — (online book) — Patterns adapted for Rust idioms
  - *Head First Design Patterns* — Freeman & Robson (2004) — Accessible introduction with visual explanations

- **Papers & Articles:**
  - [Rust Design Patterns (unofficial book)](https://rust-unofficial.github.io/patterns/) — Idiomatic Rust patterns
  - [Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/) — Deep dive into typestate
  - [Newtype Pattern in Rust](https://doc.rust-lang.org/rust-by-example/generics/new_types.html) — Rust by Example

- **Crates:**
  - [derive_builder](https://crates.io/crates/derive_builder) — Auto-generate Builder pattern implementations
  - [typed-builder](https://crates.io/crates/typed-builder) — Compile-time checked builder pattern
