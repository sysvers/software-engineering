# Modular Monolith

## Overview

A modular monolith is a single deployable application with strict internal module boundaries. It provides the organizational benefits of microservices -- independent teams, clear ownership, encapsulated domains -- without the operational complexity of distributed systems.

The modular monolith is not a stepping stone or a compromise. It is a first-class architectural pattern used by some of the most successful software companies in the world.

---

## Why Start Here

### The Distributed Systems Tax

Microservices impose a tax that teams pay whether they need the benefits or not:

- **Network reliability**: Every inter-service call can fail due to network issues. You need retries, circuit breakers, timeouts, and fallback strategies.
- **Data consistency**: Without shared transactions, you need sagas, compensating transactions, or eventual consistency patterns.
- **Operational overhead**: Each service needs its own CI/CD pipeline, monitoring, alerting, log aggregation, and deployment strategy.
- **Debugging complexity**: A single user request may traverse 10 services. Distributed tracing (Jaeger, Zipkin) becomes mandatory, not optional.
- **Service discovery**: Services need to find each other. This requires a registry (Consul, Eureka) or DNS-based discovery.
- **Schema evolution**: Changing an API contract between services requires coordinated deployment or versioning strategies.

For a team of 3-20 engineers, this tax often exceeds the benefits. The modular monolith eliminates all of these problems while preserving clean architecture.

### The False Dichotomy

The choice is not "tangled monolith vs. clean microservices." There is a middle path:

```
Tangled Monolith → Modular Monolith → Microservices
     (bad)             (good)            (sometimes good)
```

Most teams that jump to microservices are fleeing the pain of a tangled monolith. The modular monolith solves the tangling problem without introducing distributed system complexity.

---

## Module Boundaries

### What Makes a Good Module

A module in a modular monolith corresponds to a bounded context from DDD. Each module:

1. **Owns its domain model**: The module defines its own entities, value objects, and business rules.
2. **Has a public API**: Other modules interact only through explicitly exported functions, traits, or message types.
3. **Hides implementation details**: Internal data structures, database tables, and helper functions are not accessible from outside.
4. **Owns its data**: Each module manages its own database tables. No module directly queries another module's tables.

### Rust Project Structure

```
src/
├── main.rs                        # Composition root: wires modules together
├── shared/                        # Shared kernel (minimal!)
│   ├── mod.rs
│   ├── types.rs                   # Shared IDs, Money, DateTime wrappers
│   └── events.rs                  # Cross-module event definitions
│
├── orders/                        # Orders module
│   ├── mod.rs                     # Public API (re-exports only public items)
│   ├── domain/
│   │   ├── order.rs               # Order aggregate
│   │   ├── order_item.rs          # Internal entity
│   │   └── events.rs              # Order domain events
│   ├── application/
│   │   └── order_service.rs       # Use cases
│   ├── infrastructure/
│   │   └── pg_order_repo.rs       # PostgreSQL repository
│   └── api/
│       └── handlers.rs            # HTTP handlers for orders
│
├── inventory/                     # Inventory module
│   ├── mod.rs                     # Public API
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── api/
│
└── payments/                      # Payments module
    ├── mod.rs                     # Public API
    ├── domain/
    ├── application/
    ├── infrastructure/
    └── api/
```

### Enforcing Module Visibility in Rust

Rust's module system is a natural fit for enforcing boundaries. Use `pub` and `pub(crate)` deliberately:

```rust
// orders/mod.rs -- the public API of the Orders module

// Only these items are accessible to other modules
pub use self::domain::Order;
pub use self::domain::OrderId;
pub use self::domain::OrderStatus;
pub use self::application::OrderService;
pub use self::application::PlaceOrderInput;

// Everything else is private
mod domain;
mod application;
mod infrastructure;
mod api;
```

```rust
// orders/domain/order_item.rs -- internal to the Orders module

// pub(super) means visible within orders/domain/, not outside the module
pub(super) struct OrderItem {
    pub(super) product_id: ProductId,
    pub(super) quantity: u32,
    pub(super) unit_price: Money,
}
```

### Using Cargo Workspaces for Stronger Boundaries

For teams that want compiler-enforced boundaries, each module can be a separate crate in a Cargo workspace:

```toml
# Cargo.toml (workspace root)
[workspace]
members = [
    "crates/shared",
    "crates/orders",
    "crates/inventory",
    "crates/payments",
    "crates/app",          # The composition root
]
```

```toml
# crates/orders/Cargo.toml
[dependencies]
shared = { path = "../shared" }
# Note: NO dependency on inventory or payments
```

Now if the Orders crate tries to import anything from Inventory, the compiler rejects it. Boundary violations are impossible.

---

## Communication Between Modules

### Synchronous: Direct Function Calls Through Traits

The simplest approach. Modules expose traits, and the composition root wires implementations together.

```rust
// inventory/mod.rs -- public trait
pub trait InventoryChecker: Send + Sync {
    fn check_availability(&self, product_id: &ProductId, quantity: u32) -> Result<bool, InventoryError>;
}

// orders/application/order_service.rs -- uses the trait
pub struct OrderService<I: InventoryChecker, R: OrderRepository> {
    inventory: I,
    repo: R,
}

impl<I: InventoryChecker, R: OrderRepository> OrderService<I, R> {
    pub async fn place_order(&self, input: PlaceOrderInput) -> Result<Order, OrderError> {
        // Check inventory through the trait (no knowledge of inventory internals)
        for item in &input.items {
            let available = self.inventory
                .check_availability(&item.product_id, item.quantity)?;
            if !available {
                return Err(OrderError::OutOfStock(item.product_id.clone()));
            }
        }
        // ... create and save order
    }
}
```

**Trade-off**: Simple and type-safe, but creates coupling between the caller and the callee's availability. If inventory checking becomes slow, it blocks order placement.

### Asynchronous: In-Process Events

Modules communicate through events within the same process. This provides looser coupling than direct calls.

```rust
// shared/events.rs
pub enum DomainEvent {
    OrderPlaced { order_id: OrderId, items: Vec<(ProductId, u32)> },
    PaymentReceived { order_id: OrderId, amount: Money },
    InventoryReserved { order_id: OrderId },
}

// A simple in-process event bus
pub struct EventBus {
    handlers: HashMap<String, Vec<Box<dyn EventHandler>>>,
}

impl EventBus {
    pub async fn publish(&self, event: DomainEvent) -> Result<(), EventError> {
        let event_type = event.type_name();
        if let Some(handlers) = self.handlers.get(event_type) {
            for handler in handlers {
                handler.handle(&event).await?;
            }
        }
        Ok(())
    }
}

// inventory module subscribes to OrderPlaced
struct ReserveStockHandler { /* ... */ }

impl EventHandler for ReserveStockHandler {
    async fn handle(&self, event: &DomainEvent) -> Result<(), EventError> {
        if let DomainEvent::OrderPlaced { order_id, items } = event {
            // Reserve stock for each item
            for (product_id, quantity) in items {
                self.inventory_service.reserve(product_id, *quantity).await?;
            }
        }
        Ok(())
    }
}
```

**Trade-off**: Looser coupling (order module does not know about inventory module), but harder to debug (event flow is implicit, not visible in function signatures).

### Choosing Between Synchronous and Asynchronous

| Use synchronous when... | Use asynchronous when... |
|------------------------|-------------------------|
| The caller needs an immediate answer | The caller does not need a response |
| Failure should prevent the operation | Failure can be handled later (retry, compensate) |
| The operation is fast | The operation is slow or can be deferred |
| Debugging simplicity matters | Decoupling matters more than debuggability |

In practice, most modular monoliths use a mix. Order placement might synchronously check inventory (must know if items are available) but asynchronously send confirmation emails (can retry later).

---

## Enforcing Boundaries

### Automated Boundary Checks

Beyond Rust's module system, teams can add automated enforcement:

**Architecture tests** (using custom test utilities):

```rust
#[test]
fn orders_module_does_not_depend_on_payments_internals() {
    // Parse the source files in orders/ and verify no imports from payments::infrastructure
    let violations = find_imports("src/orders/", "crate::payments::infrastructure");
    assert!(violations.is_empty(), "Orders module must not access payments internals: {:?}", violations);
}
```

**Dependency analysis in CI**: Run `cargo tree` or custom scripts to verify that module crates only depend on allowed peers.

**Code review checklist**: Every PR that adds a cross-module dependency should be flagged for architect review.

### Database Boundary Enforcement

Each module should own its schema. Enforce this by:

1. **Schema prefixes**: `orders_`, `inventory_`, `payments_` table prefixes. A linting rule flags any query that touches tables from another module.
2. **Separate schemas**: PostgreSQL schemas (`orders.orders`, `inventory.products`). Module database users have permissions only on their own schema.
3. **Separate databases**: The strongest boundary. Each module has its own database connection. This also makes future microservice extraction trivial.

---

## Path to Microservices

A well-structured modular monolith makes microservice extraction straightforward:

1. **Module becomes service**: The module already has a public API (trait). Replace in-process calls with HTTP/gRPC calls behind the same trait.
2. **Database already separated**: If each module owns its schema, extracting the database is just pointing the new service at its own database instance.
3. **Events already in place**: If modules communicate via events, switch from in-process event bus to a message broker (Kafka, RabbitMQ, NATS).
4. **Extract incrementally**: Extract one module at a time. The rest of the monolith continues working. No big-bang rewrite.

### When to Extract

Extract a module into a microservice when you have a concrete reason:

- **Independent scaling**: The module needs 10x more resources than others
- **Independent deployment**: The module changes 10x more frequently and deploys are risky
- **Technology mismatch**: The module would benefit from a different language/runtime (e.g., ML model serving in Python)
- **Team autonomy**: The team owning the module is blocked by monolith release cycles

Do **not** extract because:
- "Microservices are best practice" (they are a trade-off, not a best practice)
- "We might need to scale someday" (premature optimization)
- "The monolith is messy" (fix the mess; extracting a messy module creates a messy service)

---

## Real-World Examples

### Shopify

Shopify operates one of the largest Ruby on Rails monoliths in the world. By 2019, it had grown to millions of lines of code with hundreds of engineers. Instead of a microservices rewrite, they invested in **componentization**:

- **300+ components** (modules) within the monolith, each with defined boundaries
- **Packwerk**: An open-source tool they built to enforce component boundaries in Ruby. It statically analyzes code to detect unauthorized cross-component references.
- **Separate database schemas** per component
- **Component-level CI**: Tests run only for affected components, reducing CI time

Key insight from Shopify: "The problem was never that we had a monolith. The problem was that we had an unstructured monolith." Modularization solved the structural problem without distributed system complexity.

### Basecamp / HEY

DHH (David Heinemeier Hansson), creator of Ruby on Rails and CTO of Basecamp, coined "The Majestic Monolith." Basecamp and HEY (their email product) run as a single Rails application serving millions of users.

Their argument:
- A 10-person team does not need microservices
- Operational simplicity (one thing to deploy, one thing to monitor) is worth more than theoretical scalability
- Most performance problems are solved with caching, not architectural splits
- Developer happiness matters: working in a monolith is simpler and faster

HEY processes millions of emails daily from a single Rails application with a handful of background job processors.

### Gusto (Payroll/HR Platform)

Gusto started as a monolithic Rails application. As they grew to hundreds of engineers, they invested in modularization:
- Domain-aligned modules (payroll, benefits, hiring, compliance)
- Module interfaces enforced through code review and automated checks
- Selective extraction: only the payroll calculation engine was extracted as a separate service because it had fundamentally different scaling and reliability requirements

### GitLab

GitLab is a well-known modular monolith. The entire GitLab platform (CI/CD, issue tracking, code review, package registry, security scanning) runs as a single Rails application. With over 1000 contributors, they use:
- Domain-aligned bounded contexts
- Strict module ownership
- Automated dependency analysis
- Selective use of microservices only where necessary (e.g., Gitaly for Git operations due to performance requirements)

---

## Common Mistakes

### Mistake 1: Shared Database Without Boundaries

Allowing all modules to query all tables destroys encapsulation. When the Payments module directly reads from `orders` tables, any schema change in Orders requires coordinating with Payments. This is the distributed monolith problem but within a single codebase.

### Mistake 2: Module Boundaries That Do Not Match Domain Boundaries

Drawing module lines along technical layers (all controllers in one module, all repositories in another) rather than business domains. This creates the same problems as layered architecture: changes cross modules.

### Mistake 3: Not Enforcing Boundaries

Writing clean module boundaries at the start and then letting them erode under deadline pressure. Without automated enforcement (compiler checks, CI checks, architecture tests), boundaries decay. "Just this once, I'll import directly" becomes the norm.

### Mistake 4: Premature Extraction

Extracting a module to a microservice before the module boundaries are stable. If you are still discovering where the domain boundaries lie, doing that discovery across a network boundary is 10x harder.

### Mistake 5: Over-Sharing the "Shared" Module

The shared/common module grows to include business logic, utilities, and types that belong in specific modules. The shared module should contain only truly cross-cutting concerns: basic types (IDs, Money), event definitions, and authentication primitives.

---

## Further Reading

- [Deconstructing the Monolith](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity) -- Shopify Engineering
- [The Majestic Monolith](https://m.signalvnoise.com/the-majestic-monolith/) -- DHH
- *Modular Monoliths* -- Simon Brown (talk)
- *Building Evolutionary Architectures* -- Neal Ford, Rebecca Parsons, Patrick Kua (2017)
- *Monolith to Microservices* -- Sam Newman (2019)
