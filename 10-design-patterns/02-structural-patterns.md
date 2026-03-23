# Structural Patterns

Structural patterns deal with composing structs and traits to form larger, more capable structures. They answer the question: "How do I combine existing pieces into something new without modifying the pieces themselves?" In Rust, structural patterns lean heavily on trait implementation, generics, and the `Deref` / `AsRef` patterns native to the language.

---

## Adapter Pattern

### Problem It Solves

You have two components with incompatible interfaces. One is a third-party library you cannot modify. The other is your application's trait that all payment processors (or loggers, or storage backends) must implement. You need a bridge between them without changing either side.

### Rust Implementation

```rust
// External library -- you cannot modify this
struct LegacyPaymentProcessor;

impl LegacyPaymentProcessor {
    fn process_usd_cents(&self, amount: u64, card: &str) -> bool {
        // Legacy implementation that returns true/false
        println!("Legacy: charging {amount} cents to card {card}");
        true
    }
}

// Your application's trait
#[derive(Debug)]
enum PaymentError {
    Declined,
    InvalidAmount,
    NetworkError(String),
}

trait PaymentGateway {
    fn charge(&self, amount_dollars: f64, card_number: &str) -> Result<TransactionId, PaymentError>;
}

// The adapter -- wraps the legacy processor, implements your trait
struct LegacyPaymentAdapter {
    processor: LegacyPaymentProcessor,
}

impl LegacyPaymentAdapter {
    fn new() -> Self {
        Self {
            processor: LegacyPaymentProcessor,
        }
    }
}

impl PaymentGateway for LegacyPaymentAdapter {
    fn charge(&self, amount_dollars: f64, card_number: &str) -> Result<TransactionId, PaymentError> {
        if amount_dollars <= 0.0 {
            return Err(PaymentError::InvalidAmount);
        }

        let cents = (amount_dollars * 100.0).round() as u64;

        if self.processor.process_usd_cents(cents, card_number) {
            Ok(TransactionId::new())
        } else {
            Err(PaymentError::Declined)
        }
    }
}

// Now the rest of your code works with PaymentGateway, unaware of the legacy API
fn checkout(gateway: &dyn PaymentGateway, total: f64, card: &str) -> Result<(), PaymentError> {
    let tx = gateway.charge(total, card)?;
    println!("Transaction {tx:?} completed");
    Ok(())
}
```

### Real-World Use Cases

- **Database abstraction layers** -- ORMs like Diesel wrap database-specific drivers (libpq, libmysqlclient) behind a unified query interface. Each driver adapter translates the generic query representation into database-specific wire protocol.
- **Cloud provider SDKs** -- Applications that support multiple cloud providers (AWS S3, Google Cloud Storage, Azure Blob) define a `StorageBackend` trait and write adapters for each provider's SDK.
- **Serialization format adapters** -- `serde` itself is an adapter architecture. Each format (JSON, YAML, TOML, MessagePack) provides adapters that translate between `serde`'s generic data model and the format-specific representation.
- **FFI wrappers** -- Rust crates that wrap C libraries (openssl, sqlite3, zlib) are adapters. The C API has one shape; the Rust wrapper presents an idiomatic Rust API.

### Pitfalls

1. **Leaky abstraction** -- The adapter hides the legacy interface, but error semantics may not map cleanly. A boolean `true/false` from the legacy system loses information compared to a `Result` with error variants.
2. **Performance overhead** -- Type conversions in the adapter (dollars to cents, string encoding changes) have a cost. For hot paths, measure.
3. **Adapter proliferation** -- If you need adapters for every pair of interfaces, your architecture may need a rethink. Consider defining a canonical interface early.
4. **Two-way adapters** -- If both sides need to call each other through adapters, the complexity compounds. Prefer one-directional adaptation.

### When to Use / When to Avoid

**Use when:**
- Integrating a third-party library whose interface does not match yours
- Migrating from one implementation to another (wrap the old, then the new)
- You need to test against a mock that implements your trait

**Avoid when:**
- You control both interfaces and can simply align them
- The adaptation is trivial (just renaming a method) -- consider a type alias or `Deref` instead

---

## Decorator Pattern

### Problem It Solves

You want to add behavior to an object (logging, caching, metrics, retries) without modifying its code. You want these behaviors to be composable -- stack them in any combination.

### Rust Implementation

```rust
trait Logger: Send + Sync {
    fn log(&self, message: &str);
}

// Base implementation
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

impl<L: Logger> TimestampLogger<L> {
    fn new(inner: L) -> Self {
        Self { inner }
    }
}

impl<L: Logger> Logger for TimestampLogger<L> {
    fn log(&self, message: &str) {
        let now = chrono::Utc::now().format("%Y-%m-%d %H:%M:%S");
        self.inner.log(&format!("[{now}] {message}"));
    }
}

// Decorator: adds log level prefix
struct LevelLogger<L: Logger> {
    inner: L,
    level: String,
}

impl<L: Logger> LevelLogger<L> {
    fn new(inner: L, level: &str) -> Self {
        Self { inner, level: level.to_string() }
    }
}

impl<L: Logger> Logger for LevelLogger<L> {
    fn log(&self, message: &str) {
        self.inner.log(&format!("[{}] {message}", self.level));
    }
}

// Decorator: filters messages below a threshold
struct FilterLogger<L: Logger> {
    inner: L,
    min_level: u8,
    current_level: u8,
}

impl<L: Logger> Logger for FilterLogger<L> {
    fn log(&self, message: &str) {
        if self.current_level >= self.min_level {
            self.inner.log(message);
        }
    }
}

// Composable -- stack decorators in any order
let logger = TimestampLogger::new(
    LevelLogger::new(
        ConsoleLogger,
        "INFO",
    ),
);
logger.log("Server started on port 8080");
// Output: [2024-03-15 10:30:00] [INFO] Server started on port 8080
```

### A More Practical Example: HTTP Middleware

```rust
trait HttpHandler: Send + Sync {
    fn handle(&self, req: &Request) -> Response;
}

struct AppHandler;
impl HttpHandler for AppHandler {
    fn handle(&self, req: &Request) -> Response {
        Response::ok("Hello, world!")
    }
}

// Decorator: request logging
struct LoggingMiddleware<H: HttpHandler> {
    inner: H,
}

impl<H: HttpHandler> HttpHandler for LoggingMiddleware<H> {
    fn handle(&self, req: &Request) -> Response {
        let start = Instant::now();
        let response = self.inner.handle(req);
        let elapsed = start.elapsed();
        println!("{} {} -> {} ({:?})", req.method, req.path, response.status, elapsed);
        response
    }
}

// Decorator: authentication check
struct AuthMiddleware<H: HttpHandler> {
    inner: H,
    secret: String,
}

impl<H: HttpHandler> HttpHandler for AuthMiddleware<H> {
    fn handle(&self, req: &Request) -> Response {
        match req.header("Authorization") {
            Some(token) if verify_token(token, &self.secret) => self.inner.handle(req),
            _ => Response::unauthorized("Invalid or missing token"),
        }
    }
}

// Stack: logging wraps auth wraps app
let handler = LoggingMiddleware {
    inner: AuthMiddleware {
        inner: AppHandler,
        secret: "my-secret".to_string(),
    },
};
```

### Real-World Use Cases

- **tower::Layer** -- The `tower` crate (used by axum, tonic, hyper) is built entirely on the decorator pattern. Each `Layer` wraps a `Service`, adding behavior like timeouts, rate limiting, load balancing, or retries. Layers compose by stacking.
- **tracing spans** -- The `tracing` crate decorates function calls with structured context. Each span adds metadata without modifying the instrumented code.
- **std::io::BufReader / BufWriter** -- Rust's standard library decorates raw `Read`/`Write` implementors with buffering. `BufReader::new(file)` wraps a `File` to add buffered reads.
- **Compression streams** -- `flate2::read::GzDecoder` wraps any `Read` to add decompression transparently.

### Pitfalls

1. **Deep nesting** -- Five decorators deep becomes hard to debug. Stack traces show every wrapper layer.
2. **Type complexity** -- `TimestampLogger<LevelLogger<FilterLogger<ConsoleLogger>>>` is a mouthful. Use `Box<dyn Logger>` to erase the type when the full type is not needed.
3. **Order matters** -- `Timestamp(Level(console))` produces `[time] [LEVEL] msg` while `Level(Timestamp(console))` produces `[LEVEL] [time] msg`. Document the expected order.
4. **Performance** -- Each decorator is an extra function call. In tight loops, consider combining behaviors into a single implementation.

### When to Use / When to Avoid

**Use when:**
- Behaviors are orthogonal and should be composable (logging + auth + caching)
- You want to add functionality to types you do not own
- The same base type needs different behavior combinations in different contexts

**Avoid when:**
- There is only one combination of behaviors -- just implement them directly
- The decorator chain is always the same -- a single combined implementation is simpler
- Performance is critical and the indirection is measurable

---

## Facade Pattern

### Problem It Solves

A subsystem has many components with complex interactions. Most callers only need a few high-level operations. The facade provides a simplified interface that hides the complexity, while still allowing direct access to subsystem components when needed.

### Rust Implementation

```rust
// Complex subsystem components
struct CpuMonitor {
    // reads /proc/stat, calculates usage
}

impl CpuMonitor {
    fn usage_percent(&self) -> f64 { /* ... */ 45.2 }
    fn core_count(&self) -> usize { /* ... */ 8 }
    fn per_core_usage(&self) -> Vec<f64> { /* ... */ vec![] }
    fn load_average(&self) -> (f64, f64, f64) { /* ... */ (1.2, 0.8, 0.5) }
}

struct MemoryMonitor {
    // reads /proc/meminfo
}

impl MemoryMonitor {
    fn total_bytes(&self) -> u64 { /* ... */ 16_000_000_000 }
    fn available_bytes(&self) -> u64 { /* ... */ 8_000_000_000 }
    fn available_percent(&self) -> f64 { /* ... */ 50.0 }
    fn swap_used_bytes(&self) -> u64 { /* ... */ 0 }
}

struct DiskMonitor {
    // reads filesystem stats
}

impl DiskMonitor {
    fn free_bytes(&self, mount: &str) -> u64 { /* ... */ 100_000_000_000 }
    fn free_percent(&self, mount: &str) -> f64 { /* ... */ 45.0 }
    fn iops(&self, device: &str) -> u64 { /* ... */ 1200 }
}

struct NetworkMonitor {
    // checks connectivity, measures latency
}

impl NetworkMonitor {
    fn is_reachable(&self, host: &str) -> bool { /* ... */ true }
    fn latency_ms(&self, host: &str) -> Option<f64> { /* ... */ Some(12.5) }
    fn bandwidth_mbps(&self) -> f64 { /* ... */ 940.0 }
}

// Facade -- simple interface for the common case
pub struct SystemHealth {
    cpu: CpuMonitor,
    memory: MemoryMonitor,
    disk: DiskMonitor,
    network: NetworkMonitor,
}

#[derive(Debug)]
pub struct HealthReport {
    pub healthy: bool,
    pub cpu_usage: f64,
    pub memory_available_percent: f64,
    pub disk_free_percent: f64,
    pub network_reachable: bool,
    pub issues: Vec<String>,
}

impl SystemHealth {
    pub fn new() -> Self {
        Self {
            cpu: CpuMonitor,
            memory: MemoryMonitor,
            disk: DiskMonitor,
            network: NetworkMonitor,
        }
    }

    /// Simple yes/no health check -- most callers only need this
    pub fn is_healthy(&self) -> bool {
        self.cpu.usage_percent() < 90.0
            && self.memory.available_percent() > 10.0
            && self.disk.free_percent("/") > 5.0
            && self.network.is_reachable("8.8.8.8")
    }

    /// Detailed report for dashboards and alerts
    pub fn report(&self) -> HealthReport {
        let mut issues = Vec::new();
        let cpu = self.cpu.usage_percent();
        let mem = self.memory.available_percent();
        let disk = self.disk.free_percent("/");
        let net = self.network.is_reachable("8.8.8.8");

        if cpu > 90.0 { issues.push(format!("CPU usage critical: {cpu:.1}%")); }
        if mem < 10.0 { issues.push(format!("Memory low: {mem:.1}% available")); }
        if disk < 5.0 { issues.push(format!("Disk space low: {disk:.1}% free")); }
        if !net { issues.push("Network unreachable".to_string()); }

        HealthReport {
            healthy: issues.is_empty(),
            cpu_usage: cpu,
            memory_available_percent: mem,
            disk_free_percent: disk,
            network_reachable: net,
            issues,
        }
    }

    /// Escape hatch: access subsystems directly for advanced use
    pub fn cpu(&self) -> &CpuMonitor { &self.cpu }
    pub fn memory(&self) -> &MemoryMonitor { &self.memory }
}
```

### Real-World Use Cases

- **Rust's `std::fs`** -- Functions like `fs::read_to_string()` and `fs::write()` are facades over `File::open()`, `BufReader`, `Read::read_to_string()`, and `Write::write_all()`. Most users never touch the lower-level APIs.
- **Web framework routers** -- Axum's `Router` is a facade over hyper's connection handling, tower's service composition, and tokio's async runtime. You call `.route()` and `.with_state()` without managing any of that directly.
- **rusoto / aws-sdk-rust** -- The high-level client methods (`s3.put_object()`) are facades over HTTP request construction, signing, serialization, retry logic, and connection pooling.
- **Kubernetes client libraries** -- `kube-rs` provides high-level `Api<Pod>` methods that hide the complexity of REST calls, authentication, pagination, and watch streams.

### Pitfalls

1. **Facade becomes a God Object** -- If the facade grows to expose every subsystem method, it is no longer simplifying anything. Keep facades focused.
2. **Hiding too much** -- If advanced users cannot access the underlying subsystems, the facade becomes a cage. Provide escape hatches.
3. **Stale facade** -- When subsystems evolve, the facade must be updated. If it lags behind, users bypass it, defeating the purpose.

### When to Use / When to Avoid

**Use when:**
- A subsystem has 3+ components that callers typically use together
- 80% of callers need 20% of the subsystem's functionality
- You want to provide a stable public API while the internals evolve

**Avoid when:**
- The subsystem is already simple -- a facade over one component is unnecessary
- Every caller needs different subsystem combinations -- the facade cannot serve all of them

---

## Proxy Pattern

### Problem It Solves

You want to control access to an object -- adding lazy initialization, access control, caching, or remote communication -- without changing the object's interface. The proxy stands in for the real object, intercepting calls.

### Rust Implementation

```rust
trait Database: Send + Sync {
    fn query(&self, sql: &str) -> Result<Vec<Row>, DbError>;
    fn execute(&self, sql: &str) -> Result<u64, DbError>;
}

// Real database connection
struct PostgresDb {
    pool: PgPool,
}

impl Database for PostgresDb {
    fn query(&self, sql: &str) -> Result<Vec<Row>, DbError> {
        self.pool.query(sql)
    }
    fn execute(&self, sql: &str) -> Result<u64, DbError> {
        self.pool.execute(sql)
    }
}

// Proxy: caching layer
struct CachingProxy<D: Database> {
    inner: D,
    cache: Mutex<HashMap<String, (Instant, Vec<Row>)>>,
    ttl: Duration,
}

impl<D: Database> CachingProxy<D> {
    fn new(inner: D, ttl: Duration) -> Self {
        Self {
            inner,
            cache: Mutex::new(HashMap::new()),
            ttl,
        }
    }
}

impl<D: Database> Database for CachingProxy<D> {
    fn query(&self, sql: &str) -> Result<Vec<Row>, DbError> {
        // Check cache first
        {
            let cache = self.cache.lock().unwrap();
            if let Some((inserted, rows)) = cache.get(sql) {
                if inserted.elapsed() < self.ttl {
                    return Ok(rows.clone());
                }
            }
        }

        // Cache miss -- query the real database
        let rows = self.inner.query(sql)?;

        // Store in cache
        {
            let mut cache = self.cache.lock().unwrap();
            cache.insert(sql.to_string(), (Instant::now(), rows.clone()));
        }

        Ok(rows)
    }

    fn execute(&self, sql: &str) -> Result<u64, DbError> {
        // Writes bypass cache and invalidate it
        let mut cache = self.cache.lock().unwrap();
        cache.clear();
        drop(cache);
        self.inner.execute(sql)
    }
}

// Proxy: access control
struct AuthProxy<D: Database> {
    inner: D,
    allowed_users: HashSet<String>,
}

impl<D: Database> Database for AuthProxy<D> {
    fn query(&self, sql: &str) -> Result<Vec<Row>, DbError> {
        let current_user = get_current_user();
        if !self.allowed_users.contains(&current_user) {
            return Err(DbError::AccessDenied(current_user));
        }
        self.inner.query(sql)
    }

    fn execute(&self, sql: &str) -> Result<u64, DbError> {
        let current_user = get_current_user();
        if !self.allowed_users.contains(&current_user) {
            return Err(DbError::AccessDenied(current_user));
        }
        self.inner.execute(sql)
    }
}

// Compose proxies
let db = CachingProxy::new(
    AuthProxy {
        inner: PostgresDb::connect("postgres://localhost/mydb")?,
        allowed_users: HashSet::from(["admin".into(), "service".into()]),
    },
    Duration::from_secs(60),
);
```

### Real-World Use Cases

- **Smart pointers in Rust** -- `Box<T>`, `Rc<T>`, `Arc<T>`, and `MutexGuard<T>` are all proxies. They implement `Deref` to provide transparent access to the inner value while adding heap allocation, reference counting, or synchronization.
- **Lazy initialization** -- `once_cell::sync::Lazy` is a proxy that defers initialization until first access.
- **Connection pooling** -- `r2d2` and `deadpool` return proxy connections from the pool. The proxy looks like a real connection but returns itself to the pool on drop.
- **gRPC clients** -- `tonic` generated clients are proxies that serialize method calls into protobuf messages and send them over the network.

### Pitfalls

1. **Transparency illusion** -- The proxy behaves like the real object, but edge cases differ (latency, cache staleness, access denied errors). Document the differences.
2. **Proxy chains** -- Multiple proxies stacked can make debugging difficult. Log at each proxy level to trace the call path.
3. **Lifetime complexity** -- In Rust, a proxy that borrows the inner object ties their lifetimes. Prefer owned proxies (`inner: D`) over borrowed (`inner: &D`) unless you have a specific reason.

### When to Use / When to Avoid

**Use when:**
- You need lazy initialization of expensive resources
- Access control must be enforced at the interface level
- Caching can be added transparently to an existing interface
- Remote objects need local stand-ins (RPC, network proxies)

**Avoid when:**
- The "proxy" does nothing beyond delegation -- it is dead code
- The overhead of indirection is not justified by the added behavior
- Direct access to the real object is fine for your use case
