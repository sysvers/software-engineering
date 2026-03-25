# Creational Patterns

Creational patterns control how objects and structs are created. They provide flexibility in *what* gets created, *how* it is configured, and *when* it comes into existence. In Rust, these patterns interact with the ownership system in distinctive ways -- builders consume or borrow `self`, factories return trait objects or enums, and singletons must be thread-safe by design.

---

## Builder Pattern

### Problem It Solves

A struct has many fields, some required, some optional, some with sensible defaults. Constructors with 10+ parameters are unreadable, error-prone, and impossible to extend without breaking every call site. You need a way to construct complex objects step-by-step while keeping call sites readable.

### Implementation

```text
STRUCTURE HttpRequest
    url: string, method: string, headers: list of (string, string)
    body: optional string, timeout_ms: unsigned integer
    follow_redirects: boolean, max_retries: unsigned integer

STRUCTURE HttpRequestBuilder
    (same fields as HttpRequest)

    FUNCTION NEW(url: string) -> HttpRequestBuilder
        RETURN HttpRequestBuilder {
            url, method: "GET", headers: [], body: None,
            timeout_ms: 30000, follow_redirects: true, max_retries: 3
        }

    FUNCTION METHOD(self, method) -> self;    self.method <- method; RETURN self
    FUNCTION HEADER(self, key, value) -> self; APPEND (key, value) TO headers; RETURN self
    FUNCTION BODY(self, body) -> self;         self.body <- Some(body); RETURN self
    FUNCTION TIMEOUT(self, ms) -> self;        self.timeout_ms <- ms; RETURN self
    FUNCTION NO_REDIRECTS(self) -> self;       self.follow_redirects <- false; RETURN self
    FUNCTION MAX_RETRIES(self, n) -> self;     self.max_retries <- n; RETURN self

    // Validates and produces the final request.
    FUNCTION BUILD(self) -> HttpRequest or BuildError
        IF self.url IS empty
            RETURN Err(BuildError::MissingField("url"))
        RETURN Ok(HttpRequest { url, method, headers, body, timeout_ms, follow_redirects, max_retries })

// Usage -- reads like a sentence
request <- HttpRequestBuilder.NEW("https://api.example.com/users")
    .METHOD("POST")
    .HEADER("Content-Type", "application/json")
    .BODY('{"name": "Alice"}')
    .TIMEOUT(5000)
    .NO_REDIRECTS()
    .BUILD()?
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

### Implementation

```text
INTERFACE NotificationSender
    FUNCTION SEND(to: string, message: string) -> void or Error
    FUNCTION NAME() -> string

STRUCTURE EmailSender { smtp_host: string }
    FUNCTION SEND(to, message) -> PRINT "Sending email to " + to + " via " + smtp_host; RETURN Ok
    FUNCTION NAME() -> "email"

STRUCTURE SmsSender { api_key: string }
    FUNCTION SEND(to, message) -> PRINT "Sending SMS to " + to; RETURN Ok
    FUNCTION NAME() -> "sms"

STRUCTURE SlackSender { webhook_url: string }
    FUNCTION SEND(to, message) -> PRINT "Posting to Slack channel " + to; RETURN Ok
    FUNCTION NAME() -> "slack"

// Factory function -- callers get an interface, never a concrete type.
FUNCTION CREATE_SENDER(config) -> NotificationSender or ConfigError
    MATCH config.channel
        CASE "email": RETURN Ok(NEW EmailSender { smtp_host: config.get("smtp_host") })
        CASE "sms":   RETURN Ok(NEW SmsSender { api_key: config.get("api_key") })
        CASE "slack":  RETURN Ok(NEW SlackSender { webhook_url: config.get("webhook_url") })
        DEFAULT:       RETURN Err(ConfigError::UnknownChannel(channel))

// Usage -- the caller does not know or care which sender it gets
sender <- CREATE_SENDER(config)?
sender.SEND("alice@example.com", "Your order shipped!")?
```

### Enum-Based Factory (Idiomatic Alternative)

When the set of variants is known at compile time, an enum factory avoids heap allocation:

```text
ENUMERATION Sender
    Email(EmailSender)
    Sms(SmsSender)
    Slack(SlackSender)

    FUNCTION CREATE(channel: string) -> Sender or ConfigError
        MATCH channel
            CASE "email": RETURN Ok(Sender::Email(DEFAULT EmailSender))
            CASE "sms":   RETURN Ok(Sender::Sms(DEFAULT SmsSender))
            CASE "slack":  RETURN Ok(Sender::Slack(DEFAULT SlackSender))
            DEFAULT:       RETURN Err(ConfigError::UnknownChannel(channel))

    FUNCTION SEND(to: string, message: string) -> void or Error
        MATCH self
            CASE Sender::Email(s): s.SEND(to, message)
            CASE Sender::Sms(s):   s.SEND(to, message)
            CASE Sender::Slack(s): s.SEND(to, message)
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

### Implementation (When You Must)

```text
STRUCTURE AppConfig
    database_url: string
    max_connections: unsigned integer
    log_level: string

GLOBAL CONFIG <- ONE-TIME initialized AppConfig

// Call once at startup. Panics if called twice.
FUNCTION INIT_CONFIG(path: string)
    config <- LOAD_CONFIG_FROM_FILE(path)
    CONFIG.SET(config)

// Access the config from anywhere. Panics if not yet initialized.
FUNCTION CONFIG() -> AppConfig reference
    RETURN CONFIG.GET()
```

Using `once_cell::sync::Lazy` for automatic initialization:

```text
GLOBAL LOGGER <- LAZY initialized locked list of strings

FUNCTION LOG_MESSAGE(msg: string)
    LOCK LOGGER
    APPEND msg TO LOGGER
```

### The Better Alternative: Dependency Injection

```text
STRUCTURE AppState
    config: AppConfig
    db_pool: PgPool
    cache: RedisPool

// Pass dependencies explicitly -- no globals
ASYNC FUNCTION HANDLE_REQUEST(state: AppState, req: Request) -> Response
    user <- state.db_pool.QUERY("SELECT ...")
    cached <- state.cache.GET(key)
    // ...

// In MAIN(), build the state once and pass it everywhere
ASYNC FUNCTION MAIN()
    config <- LOAD_CONFIG("config.toml")
    db_pool <- PgPool.CONNECT(config.database_url)
    cache <- RedisPool.CONNECT(config.redis_url)

    state <- SHARED(AppState { config, db_pool, cache })

    // Web frameworks accept shared state
    app <- Router.NEW()
        .ROUTE("/users", GET -> LIST_USERS)
        .WITH_STATE(state)
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
