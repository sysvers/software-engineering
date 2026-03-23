# Creational Patterns

Creational patterns control how objects and structs are created. They provide flexibility in *what* gets created, *how* it is configured, and *when* it comes into existence. In Rust, these patterns interact with the ownership system in distinctive ways -- builders consume or borrow `self`, factories return trait objects or enums, and singletons must be thread-safe by design.

---

## Builder Pattern

### Problem It Solves

A struct has many fields, some required, some optional, some with sensible defaults. Constructors with 10+ parameters are unreadable, error-prone, and impossible to extend without breaking every call site. You need a way to construct complex objects step-by-step while keeping call sites readable.

### Rust Implementation

```rust
pub struct HttpRequest {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout_ms: u64,
    follow_redirects: bool,
    max_retries: u32,
}

pub struct HttpRequestBuilder {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout_ms: u64,
    follow_redirects: bool,
    max_retries: u32,
}

impl HttpRequestBuilder {
    pub fn new(url: &str) -> Self {
        Self {
            url: url.to_string(),
            method: "GET".to_string(),
            headers: vec![],
            body: None,
            timeout_ms: 30_000,
            follow_redirects: true,
            max_retries: 3,
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

    pub fn no_redirects(mut self) -> Self {
        self.follow_redirects = false;
        self
    }

    pub fn max_retries(mut self, n: u32) -> Self {
        self.max_retries = n;
        self
    }

    /// Validates and produces the final request.
    /// Returns an error if required invariants are not met.
    pub fn build(self) -> Result<HttpRequest, BuildError> {
        if self.url.is_empty() {
            return Err(BuildError::MissingField("url"));
        }
        Ok(HttpRequest {
            url: self.url,
            method: self.method,
            headers: self.headers,
            body: self.body,
            timeout_ms: self.timeout_ms,
            follow_redirects: self.follow_redirects,
            max_retries: self.max_retries,
        })
    }
}

// Usage -- reads like a sentence
let request = HttpRequestBuilder::new("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .body(r#"{"name": "Alice"}"#)
    .timeout(5_000)
    .no_redirects()
    .build()?;
```

### Real-World Use Cases

- **reqwest::ClientBuilder** -- The most popular Rust HTTP client. Configures TLS, proxies, timeouts, redirect policies, and cookie stores through a builder before producing an immutable `Client`.
- **tokio::runtime::Builder** -- Configures the async runtime: thread count, thread names, I/O and timer drivers, all through a fluent API. Used by virtually every async Rust application.
- **clap::Command::new()** -- The CLI argument parser. Entire CLI definitions are built through chained method calls.
- **AWS SDK for Rust** -- Every AWS API request (S3 PutObject, DynamoDB Query, etc.) uses a builder. The SDK is auto-generated from service models, and every operation gets a builder struct.

### Pitfalls

1. **Forgetting validation in `build()`** -- The builder should validate invariants at build time, not let invalid objects escape. Return `Result` from `build()` rather than panicking.
2. **Mutable vs. consuming builders** -- Using `&mut self` lets you reuse the builder but risks partial builds. Using `self` (consuming) is safer because each builder is used exactly once.
3. **Too many required fields** -- If every field is required and has no default, the builder adds complexity without benefit. A plain constructor or struct literal may be better.
4. **No `Default` implementation** -- Consider deriving or implementing `Default` for the builder itself so users can start from sensible defaults.

### When to Use / When to Avoid

**Use when:**
- The struct has 4+ fields, especially if some are optional
- You need validation at construction time
- The construction process has steps that depend on each other
- API ergonomics matter (public-facing library code)

**Avoid when:**
- The struct has 2-3 simple required fields -- just use `Struct { field1, field2 }`
- All fields are always required with no defaults -- a `new()` function is simpler
- The struct is internal and only constructed in one place

---

## Factory Pattern

### Problem It Solves

You need to create objects of different concrete types based on runtime conditions (configuration, user input, environment). Callers should not know which concrete type they receive -- they work through a shared trait interface. This decouples creation logic from usage logic.

### Rust Implementation

```rust
use std::error::Error;

trait NotificationSender: Send + Sync {
    fn send(&self, to: &str, message: &str) -> Result<(), Box<dyn Error>>;
    fn name(&self) -> &str;
}

struct EmailSender {
    smtp_host: String,
}

impl NotificationSender for EmailSender {
    fn send(&self, to: &str, message: &str) -> Result<(), Box<dyn Error>> {
        println!("Sending email to {to} via {}: {message}", self.smtp_host);
        Ok(())
    }
    fn name(&self) -> &str { "email" }
}

struct SmsSender {
    api_key: String,
}

impl NotificationSender for SmsSender {
    fn send(&self, to: &str, message: &str) -> Result<(), Box<dyn Error>> {
        println!("Sending SMS to {to}: {message}");
        Ok(())
    }
    fn name(&self) -> &str { "sms" }
}

struct SlackSender {
    webhook_url: String,
}

impl NotificationSender for SlackSender {
    fn send(&self, to: &str, message: &str) -> Result<(), Box<dyn Error>> {
        println!("Posting to Slack channel {to}: {message}");
        Ok(())
    }
    fn name(&self) -> &str { "slack" }
}

/// Factory function -- callers get a trait object, never a concrete type.
fn create_sender(config: &NotificationConfig) -> Result<Box<dyn NotificationSender>, ConfigError> {
    match config.channel.as_str() {
        "email" => Ok(Box::new(EmailSender {
            smtp_host: config.get("smtp_host")?.to_string(),
        })),
        "sms" => Ok(Box::new(SmsSender {
            api_key: config.get("api_key")?.to_string(),
        })),
        "slack" => Ok(Box::new(SlackSender {
            webhook_url: config.get("webhook_url")?.to_string(),
        })),
        other => Err(ConfigError::UnknownChannel(other.to_string())),
    }
}

// Usage -- the caller does not know or care which sender it gets
let sender = create_sender(&config)?;
sender.send("alice@example.com", "Your order shipped!")?;
```

### Enum-Based Factory (Rust-Idiomatic Alternative)

When the set of variants is known at compile time, an enum factory avoids heap allocation:

```rust
enum Sender {
    Email(EmailSender),
    Sms(SmsSender),
    Slack(SlackSender),
}

impl Sender {
    fn create(channel: &str) -> Result<Self, ConfigError> {
        match channel {
            "email" => Ok(Sender::Email(EmailSender::default())),
            "sms" => Ok(Sender::Sms(SmsSender::default())),
            "slack" => Ok(Sender::Slack(SlackSender::default())),
            other => Err(ConfigError::UnknownChannel(other.to_string())),
        }
    }

    fn send(&self, to: &str, message: &str) -> Result<(), Box<dyn std::error::Error>> {
        match self {
            Sender::Email(s) => s.send(to, message),
            Sender::Sms(s) => s.send(to, message),
            Sender::Slack(s) => s.send(to, message),
        }
    }
}
```

### Real-World Use Cases

- **serde** -- Deserializers act as factories. `serde_json::from_str` and `serde_yaml::from_str` both produce the same target type through different concrete deserializers. The format is chosen at the call site, but the deserialization trait is shared.
- **Database drivers** -- `sqlx` supports Postgres, MySQL, and SQLite. The connection pool factory (`Pool::connect`) returns a pool typed to the specific database, but query execution uses shared traits.
- **Plugin systems** -- Applications that load plugins at runtime use factory functions to instantiate plugin trait objects from shared libraries.

### Pitfalls

1. **Panicking on unknown variants** -- Use `Result` instead of `panic!`. Unknown input is expected in production.
2. **Leaking concrete types** -- If callers can downcast the trait object to a concrete type, you have defeated the purpose. Keep concrete types private.
3. **Overuse with one variant** -- If there is only one implementation now and no plan for more, a factory is premature abstraction.
4. **Trait object overhead** -- `Box<dyn Trait>` involves heap allocation and dynamic dispatch. For hot paths, consider the enum-based approach or generics.

### When to Use / When to Avoid

**Use when:**
- The concrete type is determined at runtime (config, user input, feature flags)
- You need to swap implementations without changing callers
- You are building a plugin or extension system

**Avoid when:**
- There is only one implementation
- The type is known at compile time -- use generics instead
- Performance is critical and dynamic dispatch is measurable overhead

---

## Singleton Pattern

### Problem It Solves

You need exactly one instance of a resource shared across the application: a database connection pool, a configuration object, a logger, a metrics collector. The singleton ensures this single instance exists and provides global access to it.

### Why It Is Often an Anti-Pattern

1. **Hidden global state** -- Functions that access a singleton have an invisible dependency. Reading the function signature does not reveal it depends on global configuration.
2. **Testing difficulty** -- You cannot substitute a mock singleton easily. Tests that modify global state interfere with each other.
3. **Tight coupling** -- Every user of the singleton depends on it directly, making it hard to change the implementation.
4. **Initialization order** -- Who initializes it? When? What if initialization fails? What if it is accessed before initialization?

### Rust Implementation (When You Must)

```rust
use std::sync::OnceLock;

struct AppConfig {
    database_url: String,
    max_connections: u32,
    log_level: String,
}

static CONFIG: OnceLock<AppConfig> = OnceLock::new();

/// Call once at startup. Panics if called twice.
fn init_config(path: &str) {
    let config = load_config_from_file(path).expect("Failed to load config");
    CONFIG.set(config).expect("Config already initialized");
}

/// Access the config from anywhere. Panics if not yet initialized.
fn config() -> &'static AppConfig {
    CONFIG.get().expect("Config not initialized -- call init_config() first")
}
```

Using `once_cell::sync::Lazy` for automatic initialization:

```rust
use once_cell::sync::Lazy;
use std::sync::Mutex;

static LOGGER: Lazy<Mutex<Vec<String>>> = Lazy::new(|| {
    Mutex::new(Vec::new())
});

fn log_message(msg: &str) {
    LOGGER.lock().unwrap().push(msg.to_string());
}
```

### The Better Alternative: Dependency Injection

```rust
struct AppState {
    config: AppConfig,
    db_pool: PgPool,
    cache: RedisPool,
}

// Pass dependencies explicitly -- no globals
async fn handle_request(state: &AppState, req: Request) -> Response {
    let user = state.db_pool.query("SELECT ...").await?;
    let cached = state.cache.get(&key).await?;
    // ...
}

// In main(), build the state once and pass it everywhere
#[tokio::main]
async fn main() {
    let config = load_config("config.toml")?;
    let db_pool = PgPool::connect(&config.database_url).await?;
    let cache = RedisPool::connect(&config.redis_url).await?;

    let state = Arc::new(AppState { config, db_pool, cache });

    // Web frameworks like axum accept shared state
    let app = Router::new()
        .route("/users", get(list_users))
        .with_state(state);
}
```

### Real-World Use Cases

- **tracing subscriber** -- The tracing crate uses a global subscriber (effectively a singleton) set once at startup. This is one of the rare cases where a singleton is justified: logging needs to work everywhere, including in library code that cannot accept injected dependencies.
- **Global allocator** -- Rust's `#[global_allocator]` is a singleton by necessity. There can only be one allocator for the process.
- **once_cell / OnceLock in configuration** -- Many Rust applications use `OnceLock` for configuration that is loaded once at startup and never changes. This is the safest singleton pattern: immutable after initialization.

### Pitfalls

1. **Mutable singletons** -- If the singleton needs mutation, you need `Mutex` or `RwLock`, which adds contention in concurrent code.
2. **Initialization panics** -- If the singleton panics during initialization, the program crashes. Handle errors gracefully.
3. **Test pollution** -- Singletons persist across tests in the same process. Use dependency injection in testable code.
4. **Lazy initialization races** -- `Lazy` and `OnceLock` handle this correctly, but hand-rolled singletons often have subtle race conditions.

### When to Use / When to Avoid

**Use when:**
- The resource is truly process-wide (global allocator, logging infrastructure)
- The value is immutable after initialization
- Dependency injection is impractical (library code with no control over the call chain)

**Avoid when:**
- You can pass the dependency explicitly (which is almost always)
- The resource needs to be mocked in tests
- Multiple configurations might be needed (e.g., multiple database pools)
- The value is mutable -- prefer `Arc<Mutex<T>>` passed explicitly
