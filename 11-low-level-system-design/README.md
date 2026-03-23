# 11 - Low-Level System Design

## Concepts

### What is Low-Level System Design?

Low-level system design (LLD) focuses on the internal design of individual components: how to structure modules, define interfaces, manage state, and handle concurrency within a single service or application.

If high-level system design answers "what services do we need and how do they communicate?", low-level design answers "how do we build each service internally?"

LLD is where architecture meets code. It's the bridge between a box on an architecture diagram and the actual implementation.

### Module & Component Design

**Separation of concerns** is the core principle: each module should have one reason to change.

**Layered architecture within a service:**

```
┌─────────────────────────────────────┐
│         HTTP Handlers / API         │  ← Request parsing, response formatting
├─────────────────────────────────────┤
│         Application Logic           │  ← Use cases, orchestration, business rules
├─────────────────────────────────────┤
│         Domain Model                │  ← Core entities, value objects, domain rules
├─────────────────────────────────────┤
│         Infrastructure              │  ← Database, external APIs, message queues
└─────────────────────────────────────┘
```

**Dependency direction:** Upper layers depend on lower layers. Lower layers never import from upper layers. The domain model has zero dependencies on infrastructure — it's pure business logic.

**Example project structure in Rust:**

```
src/
├── main.rs                 # Entry point, wiring
├── api/
│   ├── mod.rs
│   ├── handlers.rs         # HTTP handlers (axum/actix)
│   ├── routes.rs           # Route definitions
│   └── dto.rs              # Request/response types (Data Transfer Objects)
├── application/
│   ├── mod.rs
│   └── order_service.rs    # Use cases: create_order, cancel_order
├── domain/
│   ├── mod.rs
│   ├── order.rs            # Order entity, business rules
│   ├── product.rs          # Product entity
│   └── errors.rs           # Domain errors
└── infrastructure/
    ├── mod.rs
    ├── database.rs          # Database connection, pool
    ├── order_repository.rs  # SQL queries for orders
    └── payment_client.rs    # External payment API client
```

**Why this matters:** When the database changes from PostgreSQL to MySQL, only `infrastructure/` changes. When business rules change, only `domain/` changes. When the API format changes, only `api/` changes. Each layer can be tested independently.

### Interface Design

Interfaces (traits in Rust) define contracts between components. Good interfaces enable flexibility, testability, and independent evolution.

**Principles of good interface design:**

**1. Program to interfaces, not implementations:**

```rust
// Bad — tightly coupled to PostgreSQL
fn create_order(pool: &PgPool, order: Order) -> Result<OrderId, sqlx::Error> { /* ... */ }

// Good — depends on an abstraction
trait OrderRepository {
    fn save(&self, order: &Order) -> Result<OrderId, RepositoryError>;
    fn find_by_id(&self, id: OrderId) -> Result<Option<Order>, RepositoryError>;
    fn find_by_user(&self, user_id: UserId) -> Result<Vec<Order>, RepositoryError>;
}

// Now you can swap implementations without changing callers
struct PostgresOrderRepo { pool: PgPool }
struct InMemoryOrderRepo { orders: HashMap<OrderId, Order> }  // For testing
```

**2. Interface Segregation — small, focused interfaces:**

```rust
// Bad — one massive trait
trait UserService {
    fn create(&self, user: NewUser) -> Result<User, Error>;
    fn find(&self, id: UserId) -> Result<Option<User>, Error>;
    fn update(&self, id: UserId, changes: UserUpdate) -> Result<User, Error>;
    fn delete(&self, id: UserId) -> Result<(), Error>;
    fn authenticate(&self, email: &str, password: &str) -> Result<Token, Error>;
    fn reset_password(&self, email: &str) -> Result<(), Error>;
    fn send_verification_email(&self, user: &User) -> Result<(), Error>;
    fn upload_avatar(&self, user: &User, image: &[u8]) -> Result<Url, Error>;
}

// Good — split by concern
trait UserRepository {
    fn save(&self, user: &User) -> Result<(), Error>;
    fn find_by_id(&self, id: UserId) -> Result<Option<User>, Error>;
}

trait AuthService {
    fn authenticate(&self, email: &str, password: &str) -> Result<Token, Error>;
    fn reset_password(&self, email: &str) -> Result<(), Error>;
}

trait AvatarService {
    fn upload(&self, user_id: UserId, image: &[u8]) -> Result<Url, Error>;
}
```

**3. Make impossible states unrepresentable:**

```rust
// Bad — caller must remember to check status before accessing fields
struct Payment {
    status: PaymentStatus,
    transaction_id: Option<String>,  // Only present if Completed
    error_message: Option<String>,   // Only present if Failed
    refund_id: Option<String>,       // Only present if Refunded
}

// Good — the type system enforces valid combinations
enum Payment {
    Pending { created_at: DateTime<Utc> },
    Completed { transaction_id: String, completed_at: DateTime<Utc> },
    Failed { error_message: String, failed_at: DateTime<Utc> },
    Refunded { refund_id: String, original_transaction_id: String },
}
```

### Data Flow Design

How data moves through your system determines its complexity, testability, and performance.

**Pipes and filters:**

```
Input → [Validate] → [Enrich] → [Transform] → [Store] → Output
```

Each step is a pure function that takes input and produces output. This is easy to test (each step independently) and extend (add a new step).

```rust
fn process_order(raw: RawOrderInput) -> Result<OrderConfirmation, OrderError> {
    let validated = validate_order(raw)?;
    let enriched = enrich_with_pricing(validated)?;
    let with_tax = calculate_tax(enriched)?;
    let stored = save_to_database(with_tax)?;
    let confirmation = send_confirmation(stored)?;
    Ok(confirmation)
}
```

**Middleware / interceptor pattern:**

Common in web frameworks. Each middleware wraps the next, adding behavior (logging, auth, rate limiting) without modifying the core handler.

```
Request → [Logging] → [Auth] → [RateLimit] → [Handler] → Response
                                                    ↓
Response ← [Logging] ← [Auth] ← [RateLimit] ← [Response]
```

### State Machine Design

Many systems have well-defined states and transitions. Modeling them explicitly prevents invalid states and makes the system predictable.

**Example: Subscription lifecycle**

```
                    ┌──────────────┐
                    │              │
         ┌─────────▼──┐      ┌───┴────────┐
    ──→  │   Trial    │──→   │   Active   │
         └─────┬──────┘      └───┬────┬───┘
               │                 │    │
               │          ┌─────▼─┐  │
               └─────────→│ Expired│  │
                          └───────┘  │
                                     │
                          ┌──────────▼──┐
                          │  Cancelled  │
                          └─────────────┘
```

```rust
enum SubscriptionState {
    Trial { expires_at: DateTime<Utc> },
    Active { plan: Plan, renews_at: DateTime<Utc> },
    Expired { expired_at: DateTime<Utc> },
    Cancelled { cancelled_at: DateTime<Utc>, reason: String },
}

impl SubscriptionState {
    fn activate(self, plan: Plan) -> Result<Self, SubscriptionError> {
        match self {
            Self::Trial { .. } => Ok(Self::Active {
                plan,
                renews_at: Utc::now() + Duration::days(30),
            }),
            _ => Err(SubscriptionError::InvalidTransition(
                "Can only activate from trial"
            )),
        }
    }

    fn cancel(self, reason: String) -> Result<Self, SubscriptionError> {
        match self {
            Self::Trial { .. } | Self::Active { .. } => Ok(Self::Cancelled {
                cancelled_at: Utc::now(),
                reason,
            }),
            _ => Err(SubscriptionError::InvalidTransition(
                "Cannot cancel expired or already cancelled subscription"
            )),
        }
    }
}
```

### Concurrency Design

Designing for concurrent access is critical in any system handling multiple requests simultaneously.

**Shared-nothing architecture:**
Each request handler gets its own copy of the data it needs. No shared mutable state. This is the simplest and most scalable approach.

```rust
// Each request gets its own connection from the pool
async fn handle_request(pool: &PgPool) -> Result<Response, Error> {
    let conn = pool.acquire().await?;  // Gets a connection, doesn't share it
    let data = query(&conn).await?;
    Ok(Response::json(data))
}
```

**When shared state is necessary:**
Use `Arc<Mutex<T>>` or `Arc<RwLock<T>>` carefully:

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
        entry.retain(|&t| t > window_start);  // Remove expired entries

        if entry.len() < self.max_requests {
            entry.push(now);
            true
        } else {
            false
        }
    }
}
```

**Message passing over shared memory:**
When possible, use channels instead of shared state. Each component owns its data and communicates via messages.

```rust
use tokio::sync::mpsc;

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
```

### Schema Design for Specific Use Cases

**User activity feed:**
```sql
-- Fan-out on write: pre-compute each user's feed
CREATE TABLE feed_items (
    user_id     BIGINT NOT NULL,
    activity_id BIGINT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, activity_id)
);
CREATE INDEX idx_feed_user_time ON feed_items (user_id, created_at DESC);
```

**Rate limiting with sliding window:**
```sql
CREATE TABLE rate_limits (
    key         TEXT NOT NULL,
    timestamp   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_rate_key_time ON rate_limits (key, timestamp);
-- Count requests in window:
-- SELECT COUNT(*) FROM rate_limits WHERE key = ? AND timestamp > NOW() - INTERVAL '1 minute';
```

**Audit log (append-only):**
```sql
CREATE TABLE audit_log (
    id          BIGSERIAL PRIMARY KEY,
    entity_type TEXT NOT NULL,
    entity_id   BIGINT NOT NULL,
    action      TEXT NOT NULL,       -- 'created', 'updated', 'deleted'
    actor_id    BIGINT NOT NULL,
    changes     JSONB,               -- What changed
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id, created_at DESC);
```

### Design Walkthrough: URL Shortener (LLD)

**Requirements:** Shorten URLs, redirect short URLs to originals, track click counts.

**Module design:**

```rust
// domain/
pub struct ShortUrl {
    pub code: ShortCode,       // e.g., "abc123"
    pub original_url: Url,
    pub created_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
    pub click_count: u64,
}

pub struct ShortCode(String);  // Newtype for type safety

impl ShortCode {
    pub fn generate() -> Self {
        // Base62 encoding of random bytes → 7-char code
        // 62^7 = 3.5 trillion possibilities
        Self(nanoid::nanoid!(7, &ALPHABET))
    }
}

// application/
pub struct UrlService<R: UrlRepository> {
    repo: R,
}

impl<R: UrlRepository> UrlService<R> {
    pub async fn shorten(&self, original: &str) -> Result<ShortUrl, UrlError> {
        let url = Url::parse(original).map_err(|_| UrlError::InvalidUrl)?;
        let code = ShortCode::generate();
        let short_url = ShortUrl::new(code, url);
        self.repo.save(&short_url).await?;
        Ok(short_url)
    }

    pub async fn resolve(&self, code: &str) -> Result<Url, UrlError> {
        let code = ShortCode::from(code);
        let short_url = self.repo.find_by_code(&code).await?
            .ok_or(UrlError::NotFound)?;

        if short_url.is_expired() {
            return Err(UrlError::Expired);
        }

        self.repo.increment_clicks(&code).await?;
        Ok(short_url.original_url)
    }
}

// infrastructure/
trait UrlRepository {
    async fn save(&self, url: &ShortUrl) -> Result<(), RepositoryError>;
    async fn find_by_code(&self, code: &ShortCode) -> Result<Option<ShortUrl>, RepositoryError>;
    async fn increment_clicks(&self, code: &ShortCode) -> Result<(), RepositoryError>;
}
```

## Business Value

- **Reduced development time**: Clear internal design means developers know where new code belongs. No debating "where should this go?" for every change.
- **Independent team velocity**: Well-defined module boundaries let multiple developers work on the same service without stepping on each other.
- **Testability**: Clean interfaces and dependency injection make unit testing straightforward, reducing bugs and enabling confident refactoring.
- **Onboarding speed**: A well-structured codebase is navigable. New engineers can understand and contribute to individual modules without understanding the entire system.
- **Reduced coupling cost**: When modules are properly isolated, changing one module doesn't require changes in others — the #1 driver of software maintenance cost.

## Real-World Examples

### Shopify's Modular Monolith
Shopify runs one of the largest Ruby on Rails monoliths in the world. Rather than splitting into microservices, they invested in modular monolith design: strict module boundaries enforced by tooling, each module with its own database schema, and explicit interfaces between modules. This gave them the benefits of microservices (team independence, clear ownership) without the operational complexity.

### Rust's Type-Driven Design in Cloudflare Workers
Cloudflare Workers (their edge compute platform) uses Rust internally. They leverage Rust's type system for low-level design: newtypes for different kinds of IDs, enums for request states, and the typestate pattern for connection lifecycle management. This approach catches entire categories of bugs at compile time that would be runtime errors in other languages.

### How Uber Designs Internal Services
Uber's service design guidelines mandate: every service has a clear domain boundary, communicates via well-defined interfaces (Thrift/gRPC), and separates business logic from infrastructure. Each service is structured internally as domain → application → infrastructure layers. This consistency across thousands of services means any Uber engineer can navigate any service.

### Discord's State Machine for Voice Connections
Discord models voice connections as explicit state machines. A voice connection moves through states: Disconnected → Connecting → Connected → Reconnecting → Disconnected. Invalid transitions are impossible by design. This approach eliminated an entire class of bugs where voice connections would enter invalid states, causing dropped calls.

## Common Mistakes & Pitfalls

- **Anemic domain model** — Domain objects that are just data bags with no behavior. All logic lives in "service" classes. This scatters business rules across the codebase instead of keeping them with the data they govern.

- **Leaky abstractions** — When infrastructure details leak into domain logic (SQL queries in business logic, HTTP status codes in domain errors). Keep the domain pure.

- **Circular dependencies** — Module A imports Module B, which imports Module A. This indicates unclear boundaries. Resolve by extracting shared types into a third module or inverting the dependency with a trait.

- **Stringly-typed code** — Using `String` where a dedicated type belongs. `user_id: String` vs `user_id: UserId` — the second prevents entire categories of bugs.

- **Missing error types** — Using `String` for errors or a single catch-all error type. Structured error types (enums) let callers handle specific failure modes.

- **Over-designing upfront** — Spending weeks designing the perfect module structure before writing code. Start simple, refactor as understanding grows.

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **Layered architecture** | Clear separation, familiar, testable | Can be rigid, ceremony for simple operations |
| **Flat structure** | Simple, fast to navigate for small projects | Breaks down as the codebase grows |
| **Trait-heavy design** | Flexible, testable, swappable implementations | More indirection, can be harder to follow |
| **Concrete types only** | Direct, easy to understand | Hard to test, hard to swap implementations |
| **Type-driven design (newtype, typestate)** | Compile-time safety, prevents entire bug classes | More boilerplate, learning curve for the team |

## When to Use / When Not to Use

**Invest in LLD when:**
- Building a service that multiple engineers will maintain
- The domain has complex business rules
- The service will live for years
- You need to test business logic independently from infrastructure

**Keep it simple when:**
- Building a CRUD API with no business logic
- Prototyping or validating an idea
- The service is small and maintained by one person
- The logic is straightforward data in → data out

## Key Takeaways

1. Low-level design bridges architecture and code. It answers "how do we structure the internals of a component?"
2. Separate concerns into layers: API → Application → Domain → Infrastructure. Dependencies point inward.
3. Design interfaces (traits) for your dependencies. This enables testing, flexibility, and independent evolution.
4. Make impossible states unrepresentable using Rust's enums. If a state combination is invalid, the type system should prevent it.
5. Model state machines explicitly. If your entity has defined states and transitions, encode them — don't scatter transition logic across the codebase.
6. Prefer message passing over shared mutable state for concurrency.
7. Start simple and refactor toward better design as complexity grows. Over-designing upfront is as costly as under-designing.

## Further Reading

- **Books:**
  - *A Philosophy of Software Design* — John Ousterhout (2018) — Focuses on reducing complexity in module and interface design
  - *Domain-Driven Design* — Eric Evans (2003) — The foundational text on modeling complex domains
  - *Implementing Domain-Driven Design* — Vaughn Vernon (2013) — Practical DDD implementation

- **Papers & Articles:**
  - [Making Impossible States Impossible](https://www.youtube.com/watch?v=IcgmSRJHu_8) — Richard Feldman's talk (Elm, but principles apply to Rust)
  - [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) — Alistair Cockburn's original article
  - [Modular Monolith at Shopify](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity) — Shopify Engineering Blog

- **Crates:**
  - [axum](https://crates.io/crates/axum) — Web framework that exemplifies clean layered design
  - [sqlx](https://crates.io/crates/sqlx) — Async SQL toolkit (used in infrastructure layer)
  - [nanoid](https://crates.io/crates/nanoid) — URL-safe unique ID generation
