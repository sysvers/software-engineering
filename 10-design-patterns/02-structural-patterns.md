# Structural Patterns

Structural patterns deal with composing structs and traits to form larger, more capable structures. They answer the question: "How do I combine existing pieces into something new without modifying the pieces themselves?" In Rust, structural patterns lean heavily on trait implementation, generics, and the `Deref` / `AsRef` patterns native to the language.

---

## Adapter Pattern

### Problem It Solves

You have two components with incompatible interfaces. One is a third-party library you cannot modify. The other is your application's trait that all payment processors (or loggers, or storage backends) must implement. You need a bridge between them without changing either side.

### Rust Implementation

```text
// External library -- you cannot modify this
STRUCTURE LegacyPaymentProcessor
    FUNCTION PROCESS_USD_CENTS(amount: unsigned integer, card: string) -> boolean
        // Legacy implementation that returns true/false
        PRINT "Legacy: charging " + amount + " cents to card " + card
        RETURN true

// Your application's interface
ENUMERATION PaymentError
    Declined, InvalidAmount, NetworkError(string)

INTERFACE PaymentGateway
    FUNCTION CHARGE(amount_dollars: float, card_number: string) -> TransactionId or PaymentError

// The adapter -- wraps the legacy processor, implements your interface
STRUCTURE LegacyPaymentAdapter
    processor: LegacyPaymentProcessor

    FUNCTION NEW() -> LegacyPaymentAdapter
        RETURN LegacyPaymentAdapter { processor: LegacyPaymentProcessor }

// Implement PaymentGateway for LegacyPaymentAdapter
    FUNCTION CHARGE(amount_dollars: float, card_number: string) -> TransactionId or PaymentError
        IF amount_dollars <= 0.0
            RETURN Err(PaymentError::InvalidAmount)
        cents <- ROUND(amount_dollars * 100.0)
        IF self.processor.PROCESS_USD_CENTS(cents, card_number)
            RETURN Ok(NEW TransactionId)
        ELSE
            RETURN Err(PaymentError::Declined)

// Now the rest of your code works with PaymentGateway, unaware of the legacy API
FUNCTION CHECKOUT(gateway: PaymentGateway, total: float, card: string) -> void or PaymentError
    tx <- gateway.CHARGE(total, card)?
    PRINT "Transaction " + tx + " completed"
    RETURN Ok
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

```text
INTERFACE Logger
    FUNCTION LOG(message: string)

// Base implementation
STRUCTURE ConsoleLogger
    FUNCTION LOG(message) -> PRINT message

// Decorator: adds timestamps
STRUCTURE TimestampLogger<L: Logger>
    inner: L
    FUNCTION LOG(message)
        now <- FORMAT_TIME(UTC_NOW(), "YYYY-MM-DD HH:MM:SS")
        self.inner.LOG("[" + now + "] " + message)

// Decorator: adds log level prefix
STRUCTURE LevelLogger<L: Logger>
    inner: L, level: string
    FUNCTION LOG(message)
        self.inner.LOG("[" + self.level + "] " + message)

// Decorator: filters messages below a threshold
STRUCTURE FilterLogger<L: Logger>
    inner: L, min_level: byte, current_level: byte
    FUNCTION LOG(message)
        IF self.current_level >= self.min_level
            self.inner.LOG(message)

// Composable -- stack decorators in any order
logger <- NEW TimestampLogger(NEW LevelLogger(ConsoleLogger, "INFO"))
logger.LOG("Server started on port 8080")
// Output: [2024-03-15 10:30:00] [INFO] Server started on port 8080
```

### A More Practical Example: HTTP Middleware

```text
INTERFACE HttpHandler
    FUNCTION HANDLE(req: Request) -> Response

STRUCTURE AppHandler
    FUNCTION HANDLE(req) -> Response.OK("Hello, world!")

// Decorator: request logging
STRUCTURE LoggingMiddleware<H: HttpHandler>
    inner: H
    FUNCTION HANDLE(req) -> Response
        start <- NOW()
        response <- self.inner.HANDLE(req)
        elapsed <- NOW() - start
        PRINT req.method + " " + req.path + " -> " + response.status + " (" + elapsed + ")"
        RETURN response

// Decorator: authentication check
STRUCTURE AuthMiddleware<H: HttpHandler>
    inner: H, secret: string
    FUNCTION HANDLE(req) -> Response
        MATCH req.HEADER("Authorization")
            CASE Some(token) IF VERIFY_TOKEN(token, self.secret): RETURN self.inner.HANDLE(req)
            DEFAULT: RETURN Response.UNAUTHORIZED("Invalid or missing token")

// Stack: logging wraps auth wraps app
handler <- LoggingMiddleware { inner: AuthMiddleware { inner: AppHandler, secret: "my-secret" } }
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

### Implementation

```text
// Complex subsystem components
STRUCTURE CpuMonitor        // reads /proc/stat, calculates usage
    FUNCTION USAGE_PERCENT() -> float         // e.g., 45.2
    FUNCTION CORE_COUNT() -> size             // e.g., 8
    FUNCTION PER_CORE_USAGE() -> list of float
    FUNCTION LOAD_AVERAGE() -> (float, float, float)

STRUCTURE MemoryMonitor     // reads /proc/meminfo
    FUNCTION TOTAL_BYTES() -> unsigned integer
    FUNCTION AVAILABLE_PERCENT() -> float

STRUCTURE DiskMonitor       // reads filesystem stats
    FUNCTION FREE_PERCENT(mount: string) -> float

STRUCTURE NetworkMonitor    // checks connectivity
    FUNCTION IS_REACHABLE(host: string) -> boolean
    FUNCTION LATENCY_MS(host: string) -> optional float

// Facade -- simple interface for the common case
STRUCTURE SystemHealth
    cpu: CpuMonitor, memory: MemoryMonitor, disk: DiskMonitor, network: NetworkMonitor

    // Simple yes/no health check -- most callers only need this
    FUNCTION IS_HEALTHY() -> boolean
        RETURN cpu.USAGE_PERCENT() < 90.0
            AND memory.AVAILABLE_PERCENT() > 10.0
            AND disk.FREE_PERCENT("/") > 5.0
            AND network.IS_REACHABLE("8.8.8.8")

    // Detailed report for dashboards and alerts
    FUNCTION REPORT() -> HealthReport
        issues <- empty list
        cpu <- self.cpu.USAGE_PERCENT()
        mem <- self.memory.AVAILABLE_PERCENT()
        disk <- self.disk.FREE_PERCENT("/")
        net <- self.network.IS_REACHABLE("8.8.8.8")

        IF cpu > 90.0 THEN APPEND "CPU usage critical" TO issues
        IF mem < 10.0 THEN APPEND "Memory low" TO issues
        IF disk < 5.0 THEN APPEND "Disk space low" TO issues
        IF NOT net THEN APPEND "Network unreachable" TO issues

        RETURN HealthReport { healthy: issues IS EMPTY, cpu_usage: cpu,
            memory_available_percent: mem, disk_free_percent: disk,
            network_reachable: net, issues }

    // Escape hatch: access subsystems directly for advanced use
    FUNCTION CPU() -> CpuMonitor reference
    FUNCTION MEMORY() -> MemoryMonitor reference
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

### Implementation

```text
INTERFACE Database
    FUNCTION QUERY(sql: string) -> list of Row or DbError
    FUNCTION EXECUTE(sql: string) -> unsigned integer or DbError

// Real database connection
STRUCTURE PostgresDb { pool: PgPool }
    FUNCTION QUERY(sql) -> self.pool.QUERY(sql)
    FUNCTION EXECUTE(sql) -> self.pool.EXECUTE(sql)

// Proxy: caching layer
STRUCTURE CachingProxy<D: Database>
    inner: D, cache: locked map of (sql -> (timestamp, rows)), ttl: Duration

    FUNCTION QUERY(sql) -> list of Row or DbError
        // Check cache first
        LOCK cache
        IF cache HAS sql AND cache[sql].timestamp.ELAPSED() < self.ttl
            RETURN Ok(cache[sql].rows)
        UNLOCK cache

        // Cache miss -- query the real database
        rows <- self.inner.QUERY(sql)?

        // Store in cache
        LOCK cache
        cache[sql] <- (NOW(), rows)
        RETURN Ok(rows)

    FUNCTION EXECUTE(sql) -> unsigned integer or DbError
        // Writes bypass cache and invalidate it
        LOCK cache; CLEAR cache; UNLOCK cache
        RETURN self.inner.EXECUTE(sql)

// Proxy: access control
STRUCTURE AuthProxy<D: Database>
    inner: D, allowed_users: set of string

    FUNCTION QUERY(sql) -> list of Row or DbError
        current_user <- GET_CURRENT_USER()
        IF current_user NOT IN self.allowed_users
            RETURN Err(DbError::AccessDenied(current_user))
        RETURN self.inner.QUERY(sql)

    FUNCTION EXECUTE(sql) -> unsigned integer or DbError
        current_user <- GET_CURRENT_USER()
        IF current_user NOT IN self.allowed_users
            RETURN Err(DbError::AccessDenied(current_user))
        RETURN self.inner.EXECUTE(sql)

// Compose proxies
db <- NEW CachingProxy(
    AuthProxy { inner: PostgresDb.CONNECT("postgres://localhost/mydb"), allowed_users: {"admin", "service"} },
    ttl: 60 seconds
)
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
