# Module & Component Design

## Core Principle: Separation of Concerns

Each module should have **one reason to change**. This is the single responsibility principle applied at the module level. When you look at a module, you should be able to describe its purpose in one sentence. If you need "and" in that sentence, the module probably does too much.

Separation of concerns operates at multiple levels:
- **Function level:** Each function does one thing.
- **Module level:** Each module owns one slice of the problem.
- **Layer level:** Each architectural layer handles one type of responsibility.

## Layered Architecture Within a Service

The most common internal structure for a non-trivial service:

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

**Dependency direction:** Upper layers depend on lower layers. Lower layers never import from upper layers. The domain model has zero dependencies on infrastructure — it is pure business logic.

### What Each Layer Does

**API layer** — Translates between the outside world and your application. Parses HTTP requests, validates input formats, serializes responses. Knows about JSON, HTTP status codes, headers. Knows nothing about databases or business rules.

**Application layer** — Orchestrates use cases. "When a user places an order: validate the cart, check inventory, charge payment, create the order, send confirmation." Each use case is a function or method. Knows about the domain but not about infrastructure details.

**Domain layer** — The heart. Contains entities (Order, User, Product), value objects (Money, EmailAddress), and domain rules ("an order cannot exceed $10,000 without manager approval"). Has zero external dependencies.

**Infrastructure layer** — Implements the details: database queries, HTTP clients for external services, message queue producers/consumers. Implements traits defined by the domain or application layers.

## Rust Project Structure

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

### Wiring It All Together

The entry point (`main.rs`) is where dependency injection happens. It constructs concrete implementations and passes them to the layers that need abstractions:

```text
ASYNC FUNCTION MAIN()
    pool ← AWAIT PG_POOL_CONNECT(database_url)
    order_repo ← NEW PostgresOrderRepo(pool)
    payment_client ← NEW StripeClient(stripe_api_key)
    order_service ← NEW OrderService(order_repo, payment_client)

    app ← NEW Router()
        .ROUTE("/orders", POST → CREATE_ORDER)
        .ROUTE("/orders/:id", GET → GET_ORDER)
        .WITH_STATE(AppState { order_service ← order_service })

    AWAIT SERVER_BIND(addr).SERVE(app)
```

No layer instantiates its own dependencies. The entry point is the only place that knows about all concrete types.

## Data Flow Design

### Pipes and Filters

```
Input → [Validate] → [Enrich] → [Transform] → [Store] → Output
```

Each step is a pure function that takes input and produces output. Easy to test (each step independently) and extend (add a new step).

```text
FUNCTION PROCESS_ORDER(raw: RawOrderInput) → Result<OrderConfirmation, OrderError>
    validated ← VALIDATE_ORDER(raw)?
    enriched ← ENRICH_WITH_PRICING(validated)?
    with_tax ← CALCULATE_TAX(enriched)?
    stored ← SAVE_TO_DATABASE(with_tax)?
    confirmation ← SEND_CONFIRMATION(stored)?
    RETURN confirmation
```

### Middleware / Interceptor Pattern

Common in web frameworks. Each middleware wraps the next, adding behavior (logging, auth, rate limiting) without modifying the core handler.

```
Request → [Logging] → [Auth] → [RateLimit] → [Handler] → Response
                                                    ↓
Response ← [Logging] ← [Auth] ← [RateLimit] ← [Response]
```

## Real-World Examples

### Shopify's Modular Monolith

Shopify runs one of the largest Ruby on Rails monoliths in the world. Rather than splitting into microservices, they invested in modular monolith design: strict module boundaries enforced by tooling, each module with its own database schema, and explicit interfaces between modules. This gave them the benefits of microservices (team independence, clear ownership) without the operational complexity.

Key lessons:
- Module boundaries must be enforced by tooling, not just convention.
- Each module owns its data — no cross-module database queries.
- Explicit public interfaces between modules make dependencies visible and manageable.

### Uber's Service Design Guidelines

Uber's service design guidelines mandate: every service has a clear domain boundary, communicates via well-defined interfaces (Thrift/gRPC), and separates business logic from infrastructure. Each service is structured internally as domain, application, and infrastructure layers. This consistency across thousands of services means any Uber engineer can navigate any service.

Key lessons:
- Consistent internal structure across all services reduces cognitive load.
- Domain boundaries should match organizational boundaries (Conway's Law).
- Standardized interface definitions (IDL) prevent ad-hoc coupling between services.

## Common Mistakes

- **Anemic domain model** — Domain objects that are just data bags with no behavior. All logic lives in "service" classes. This scatters business rules across the codebase.
- **Circular dependencies** — Module A imports Module B, which imports Module A. This indicates unclear boundaries. Resolve by extracting shared types into a third module or inverting the dependency with a trait.
- **Leaky abstractions** — When infrastructure details leak into domain logic (SQL queries in business logic, HTTP status codes in domain errors). Keep the domain pure.
- **Over-designing upfront** — Spending weeks designing the perfect module structure before writing code. Start simple, refactor as understanding grows.

## When to Invest in Module Design

**Do invest** when multiple engineers maintain the service, the domain has complex business rules, or the service will live for years.

**Keep it simple** when building a CRUD API with no business logic, prototyping, or the service is small and maintained by one person.
