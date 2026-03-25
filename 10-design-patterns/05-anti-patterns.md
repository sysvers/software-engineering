# Anti-Patterns

Anti-patterns are recurring solutions that look reasonable but create more problems than they solve. Recognizing them is as important as knowing design patterns -- you will encounter them in every codebase of sufficient age. This document covers the most common anti-patterns, how to spot them, and how to fix them.

---

## God Object

### What It Is

A single struct, class, or module that knows too much and does too much. It accumulates responsibilities over time because "it already has access to the data" or "it is easier to add it here." Eventually, every change touches this one file, every test requires instantiating this monster, and every developer has merge conflicts in it.

### How to Recognize It

- The struct has 20+ fields
- The impl block has 50+ methods spanning unrelated domains
- The file is 1000+ lines and growing
- Every new feature "naturally" lands in this module
- It is the most frequently modified file in `git log --stat`
- Testing requires mocking or initializing dozens of dependencies

### Real Example

```text
// The God Object -- this is NOT how to write code
RECORD Application
    db_pool: PgPool
    redis: RedisClient
    config: AppConfig
    email_client: SmtpClient
    payment_processor: StripeClient
    search_index: ElasticClient
    cache: Map<String, CachedValue>
    rate_limiter: RateLimiter
    metrics: MetricsCollector
    logger: Logger
    users: List<User>
    orders: List<Order>
    products: List<Product>

FUNCTION CREATE_USER(app: Application, ...)      // touches db, email, search, metrics
FUNCTION PROCESS_PAYMENT(app: Application, ...)   // touches payment, db, email, metrics
FUNCTION SEARCH_PRODUCTS(app: Application, ...)   // touches search, cache, metrics
FUNCTION GENERATE_REPORT(app: Application, ...)   // touches db, email
FUNCTION HANDLE_WEBHOOK(app: Application, ...)    // touches everything
FUNCTION SEND_NOTIFICATION(app: Application, ...) // touches email, db
FUNCTION UPDATE_INVENTORY(app: Application, ...)  // touches db, search, cache
// ... 40 more methods
```

### How to Fix It

Split along domain boundaries. Each service owns its data and dependencies:

```text
RECORD UserService
    db: PgPool
    search: ElasticClient
    email: SmtpClient

FUNCTION CREATE_USER(svc: UserService, req: CreateUserRequest) → Result<User, UserError>
FUNCTION FIND_USER(svc: UserService, id: UserId) → Result<User, UserError>
FUNCTION DEACTIVATE_USER(svc: UserService, id: UserId) → Result<Void, UserError>

RECORD OrderService
    db: PgPool
    payment: StripeClient
    email: SmtpClient

FUNCTION PLACE_ORDER(svc: OrderService, req: PlaceOrderRequest) → Result<Order, OrderError>
FUNCTION CANCEL_ORDER(svc: OrderService, id: OrderId) → Result<Void, OrderError>

RECORD ProductService
    db: PgPool
    search: ElasticClient
    cache: Cache

FUNCTION SEARCH_PRODUCTS(svc: ProductService, query: String) → Result<List<Product>, SearchError>
FUNCTION UPDATE_STOCK(svc: ProductService, id: ProductId, delta: Integer) → Result<Void, StockError>
```

Each service is testable in isolation, has a focused API, and changes to one domain do not ripple through others.

---

## Spaghetti Code

### What It Is

Code with no clear structure, where control flow jumps between modules unpredictably. Functions call each other in cycles. Module A imports B, which imports C, which imports A. There is no layering, no direction of dependency, no way to understand the code by reading it top-down.

### How to Recognize It

- Circular dependencies between modules
- A function that calls 15 other functions across 10 files
- You cannot explain the call graph of any feature without a whiteboard
- Adding a feature requires touching 8+ files in unrelated directories
- No clear "entry point" for any operation
- `grep` for a function name shows it called from everywhere

### Real Example (Conceptual)

```
// Spaghetti dependency graph -- arrows mean "calls"
handlers.rs -> auth.rs -> db.rs -> cache.rs -> handlers.rs  (cycle!)
              \-> email.rs -> db.rs -> config.rs -> email.rs (cycle!)
              \-> billing.rs -> auth.rs (cycle through handlers!)
```

Every module depends on every other module, directly or transitionally.

### How to Fix It

Establish layers with a strict dependency direction:

```
Layer 4: Handlers      (HTTP handlers, CLI commands)
    |
Layer 3: Services      (business logic, orchestration)
    |
Layer 2: Repositories  (data access, external APIs)
    |
Layer 1: Domain        (types, traits, pure functions)
```

Rules:
- Each layer may only depend on layers below it. Never upward, never sideways at the same level.
- Layer 1 (domain) has zero dependencies on infrastructure.
- Shared behavior goes in the lowest appropriate layer.

```text
// Layer 1: Domain -- pure types, no I/O
MODULE domain
    TYPE UserId = WRAPPER(Integer)
    RECORD User { id: UserId, name: String, email: String }
    INTERFACE UserRepository
        FUNCTION FIND(id: UserId) → Result<User, RepoError>
        FUNCTION SAVE(user: User) → Result<Void, RepoError>

// Layer 2: Repository -- implements domain traits with real I/O
MODULE repository
    USES domain
    RECORD PgUserRepository { pool: PgPool }
    IMPLEMENTS UserRepository FOR PgUserRepository

// Layer 3: Service -- orchestrates domain logic
MODULE service
    USES domain
    RECORD UserService<R: UserRepository> { repo: R }
    FUNCTION REGISTER(svc: UserService, name: String, email: String) → Result<User, ServiceError>

// Layer 4: Handler -- thin adapter between HTTP and service
MODULE handler
    USES service
    ASYNC FUNCTION CREATE_USER(service: UserService, req: Request) → Response
        // parse request, call service, format response
```

---

## Lava Flow

### What It Is

Dead code, unused abstractions, and deprecated modules that remain in the codebase because nobody is sure if something still depends on them. Like cooled lava, it is hardened in place -- too scary to remove, too useless to maintain. It grows over years as features are added but old code is never deleted.

### How to Recognize It

- Functions that are never called (but are `pub` "just in case")
- Modules with comments like `// TODO: remove after migration` from 3 years ago
- Traits with a single implementation that was added "for future extensibility" that never came
- Feature flags that are always on (or always off)
- Test files that are `#[ignore]`d indefinitely
- Cargo dependencies in `Cargo.toml` that nothing imports

### Real Example

```text
// This function was replaced by PROCESS_PAYMENT_V2 in 2023.
// Nobody has confirmed it is safe to delete.
// It has 200 lines of complex logic.
[DEPRECATED: "use PROCESS_PAYMENT_V2"]
FUNCTION PROCESS_PAYMENT(order: Order) → Result<Void, PaymentError>
    // ... 200 lines ...

// "Temporary" compatibility shim from the database migration.
// Marked for removal in Q1 2024. It is now Q1 2026.
FUNCTION LEGACY_USER_LOOKUP(id: Integer) → Optional<User>
    // translates old user IDs to new UUIDs
    // nobody knows if this is still called

// Abstract factory for notification senders.
// Only one implementation was ever written.
// The interface exists "in case we add more later."
INTERFACE NotificationSenderFactory
    FUNCTION CREATE(channel: String) → NotificationSender
```

### How to Fix It

1. **Delete it.** Git remembers. If you need it back, `git log` and `git show` will find it.
2. **Use tooling:**
   - `cargo udeps` -- finds unused dependencies
   - `cargo clippy` -- warns about dead code, unused variables, unreachable patterns
   - `#[warn(dead_code)]` -- Rust warns about unused private items by default; do not suppress the warning with `#[allow(dead_code)]` unless you have a concrete reason
3. **Set a removal date.** When deprecating code, add a date: `// REMOVE AFTER 2025-06-01`. Schedule a calendar reminder.
4. **Measure usage.** Add logging or metrics to suspect code. If it gets zero hits in a month, delete it.
5. **Run tests after deleting.** If tests pass, the code was dead. If they fail, you know exactly what depends on it.

---

## Golden Hammer

### What It Is

"When all you have is a hammer, everything looks like a nail." The golden hammer anti-pattern is using one familiar tool, pattern, or technology for every problem, regardless of fit. It stems from comfort and expertise bias rather than objective analysis.

### How to Recognize It

- Every service uses the same database, even when a different one would be 10x better (using Postgres for a job queue instead of Redis, or for full-text search instead of Elasticsearch)
- Every problem gets a microservice, even when a function call would suffice
- Every struct gets a Builder pattern, even 2-field structs
- Every bit of shared behavior becomes a trait, even when a plain function works
- The team chooses the same language/framework for every project regardless of requirements
- Architecture discussions end with "we always do it this way"

### Real Example

A team that knows Kubernetes well deploys everything to Kubernetes:
- A cron job that runs once a day? Kubernetes CronJob.
- A static marketing website? Kubernetes with nginx.
- A CLI tool for internal use? Containerized and deployed to Kubernetes.
- A one-off data migration script? Kubernetes Job.

None of these need container orchestration. A cron entry, a static file host, a binary on PATH, and a shell script would each be simpler, cheaper, and faster to maintain.

### In Rust Specifically

```text
// Golden hammer: making everything generic when concrete types are fine

// Over-engineered -- there will only ever be one Config type
FUNCTION LOAD_CONFIG<C: Config + Deserializable + Default>(path: String) → Result<C, ConfigError>
    // ...

// Just use the concrete type
FUNCTION LOAD_CONFIG(path: String) → Result<AppConfig, ConfigError>
    // ...
```

```text
// Golden hammer: interface objects everywhere

// Over-engineered -- there is only one logger implementation
FUNCTION PROCESS_ORDER(order: Order, logger: Logger, mailer: Mailer)  // ...

// The concrete types are fine until you actually need polymorphism
FUNCTION PROCESS_ORDER(order: Order, logger: ConsoleLogger, mailer: SmtpMailer)  // ...
```

### How to Fix It

1. **Ask "why this tool?"** before every technical decision. The answer should reference the problem, not the tool.
2. **Learn alternatives.** Golden hammers persist because teams do not know other options exist.
3. **Prototype with the "wrong" tool.** If you always use Postgres, try Redis for one project. If you always use microservices, try a monolith. The learning is valuable even if you switch back.
4. **Decision records.** Write down why you chose a technology, linking it to specific requirements. Revisit these records when requirements change.

---

## Premature Abstraction

### What It Is

Adding layers of abstraction before complexity justifies them. Creating interfaces, traits, and generic structures for code that currently has one implementation and may never need another. It is the opposite of YAGNI (You Aren't Gonna Need It).

### How to Recognize It

- A trait with exactly one implementation
- A generic function used with exactly one type
- An "AbstractFactory" that creates exactly one kind of thing
- Layers of indirection where a direct call would be clearer
- Comments like "this will be useful when we add [feature that is not planned]"
- The abstraction makes the code harder to navigate (you must jump through 3 files to find the actual logic)

### Real Example

```text
// Premature abstraction -- there is only one database and one storage backend

INTERFACE StorageBackend
    FUNCTION READ(key: String) → Result<Bytes, StorageError>
    FUNCTION WRITE(key: String, data: Bytes) → Result<Void, StorageError>
    FUNCTION DELETE(key: String) → Result<Void, StorageError>

INTERFACE StorageBackendFactory
    FUNCTION CREATE() → StorageBackend

RECORD FileStorageBackend { base_path: Path }
IMPLEMENTS StorageBackend FOR FileStorageBackend

RECORD FileStorageBackendFactory { base_path: Path }
IMPLEMENTS StorageBackendFactory FOR FileStorageBackendFactory
    FUNCTION CREATE() → StorageBackend
        RETURN NEW FileStorageBackend { base_path ← self.base_path }

// The "service" that uses all this machinery
RECORD DocumentService
    storage: StorageBackend
```

All of that could be:

```text
// Direct implementation -- add abstraction when a second backend appears
RECORD DocumentService
    base_path: Path

FUNCTION READ(svc: DocumentService, key: String) → Result<Bytes, IOError>
    RETURN FILE_READ(svc.base_path + "/" + key)

FUNCTION WRITE(svc: DocumentService, key: String, data: Bytes) → Result<Void, IOError>
    FILE_WRITE(svc.base_path + "/" + key, data)

FUNCTION DELETE(svc: DocumentService, key: String) → Result<Void, IOError>
    FILE_REMOVE(svc.base_path + "/" + key)
```

When a second storage backend is actually needed (not hypothetically), then extract the trait. At that point you have two concrete examples to inform the trait design, which produces a better abstraction than guessing up front.

### The Rule of Three

A practical heuristic: do not abstract until you have three concrete instances.

1. **First time:** Write the code directly.
2. **Second time:** Notice the duplication but tolerate it. You now have two examples to understand the pattern.
3. **Third time:** Extract the abstraction. You have enough examples to design it well.

### How to Fix Existing Premature Abstractions

1. **Inline the abstraction.** If a trait has one implementor, replace trait usage with the concrete type. If a factory creates one kind of thing, replace it with a constructor.
2. **Delete unused generics.** If a function is generic over `T` but only ever called with `String`, make it take `String`.
3. **Flatten layers.** If you must jump through `Controller -> Service -> Repository -> Adapter` and each layer has one implementation, collapse them until the indirection is justified.
4. **Measure before you abstract.** Abstractions should reduce the total complexity of the codebase. If the abstraction adds more code than it saves, it is premature.

### When Abstraction IS Justified

- You have 2+ implementations today (not hypothetically)
- The abstraction enables testing (mocking a database trait for unit tests)
- It is a public API boundary that must remain stable while internals evolve
- The abstraction reduces code duplication measurably

### When It Is Not

- "We might need this later" -- YAGNI
- "It is best practice" -- best practice depends on context
- "All the other services have this layer" -- consistency is not a reason to add unnecessary complexity
- "It makes the architecture diagram look cleaner" -- architecture diagrams are not the product
