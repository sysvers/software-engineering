# Architectural Styles

## Overview

An architectural style defines the fundamental structural pattern of a software system: how components are organized, how they communicate, and what constraints govern their relationships. Choosing the right style is one of the most consequential decisions in a project because it shapes everything from testability to team structure.

This document covers three widely-used styles in detail: Layered, Hexagonal (Ports & Adapters), and Clean Architecture.

---

## Layered Architecture

### Concept

The system is divided into horizontal layers, each with a specific responsibility. Each layer only depends on the layer directly below it. This is the most common and familiar style, dating back to the 1960s with operating system design.

```
┌─────────────────────────┐
│    Presentation Layer   │  UI, API endpoints
├─────────────────────────┤
│    Business Logic Layer │  Rules, workflows, validation
├─────────────────────────┤
│    Data Access Layer    │  Repositories, queries
├─────────────────────────┤
│    Database             │  Storage
└─────────────────────────┘
```

### How It Works

- **Presentation Layer**: Handles user input and output. In a web app, this is HTTP handlers, request parsing, and response formatting. In a CLI, it is argument parsing and terminal output.
- **Business Logic Layer**: Contains the rules and workflows that define what the system does. Order validation, pricing calculations, access control decisions.
- **Data Access Layer**: Abstracts storage. Translates between the business layer's domain objects and the database's rows/documents.
- **Database Layer**: The actual storage engine (PostgreSQL, Redis, filesystem).

Each layer calls only into the layer directly below. Presentation calls business logic, business logic calls data access, data access calls the database. Calls never skip layers and never go upward.

### Real-World Examples

**Java Spring Boot applications** are the canonical example. A typical Spring project has `@Controller` classes (presentation), `@Service` classes (business logic), and `@Repository` classes (data access). The framework enforces this layering by convention.

**Django (Python)** follows a variation: views (presentation), models with business methods (logic + data access combined), and the ORM (database). Django's "fat models" pattern intentionally merges the middle two layers.

**Enterprise banking systems** almost universally use layered architecture. COBOL mainframe applications from the 1980s often have clear presentation/logic/data separation, and modern replacements preserve this pattern because it maps well to regulatory auditing: each layer has clear responsibilities that auditors can inspect independently.

### Strengths

- **Familiarity**: Nearly every developer has worked with layered architecture. Onboarding is fast.
- **Separation of concerns**: Each layer has a clear job. Database changes don't require changing the UI layer.
- **Tooling support**: Most web frameworks (Rails, Django, Spring, ASP.NET) are built around layered patterns.
- **Audibility**: Regulators and security auditors can examine layers independently.

### Weaknesses and Pitfalls

- **Lasagna architecture**: Changes that cross concerns (e.g., adding a new field) require modifications in every layer: database migration, repository method, service method, API handler, and DTO. This creates a "change amplification" problem.
- **Pass-through layers**: Layers that add no logic but exist for "completeness" create ceremony. A service method that calls `self.repo.find_by_id(id)` and returns the result unchanged adds indirection without value.
- **Business logic leakage**: In practice, logic tends to leak upward into controllers ("just add this validation here, it's simpler") and downward into SQL queries ("the database can calculate this faster"). Over time, the business layer hollows out and logic scatters.
- **Tight coupling to the database**: Because dependencies flow downward, the business layer depends on the data access layer, which depends on the database. Changing databases means rewriting from the bottom up.

### When to Use

- CRUD-heavy applications where business logic is thin
- Small teams (1-5 developers) where simplicity matters more than flexibility
- Applications built on opinionated frameworks that enforce layering (Rails, Django)
- Systems with regulatory/audit requirements that benefit from clear layer separation

### Rust Project Structure Example

```
src/
├── main.rs
├── api/                  # Presentation layer
│   ├── mod.rs
│   ├── handlers.rs       # HTTP handlers (axum/actix)
│   └── dto.rs            # Request/response types
├── service/              # Business logic layer
│   ├── mod.rs
│   └── order_service.rs  # Business rules and workflows
├── repository/           # Data access layer
│   ├── mod.rs
│   └── order_repo.rs     # SQL queries, database interaction
└── models/               # Shared domain types
    ├── mod.rs
    └── order.rs
```

---

## Hexagonal Architecture (Ports & Adapters)

### Concept

Invented by Alistair Cockburn in 2005. The core insight is revolutionary in its simplicity: the business logic should not know how it is accessed or what infrastructure it uses. The domain sits at the center, communicating with the outside world through **ports** (abstract interfaces) and **adapters** (concrete implementations).

```
                 ┌──────────────────────┐
    HTTP ──→ [Adapter] ──→ │                      │
                           │   DOMAIN / BUSINESS   │
    CLI  ──→ [Adapter] ──→ │       LOGIC          │ ──→ [Adapter] ──→ PostgreSQL
                           │                      │
   Tests ──→ [Adapter] ──→ │   (Ports = Traits)   │ ──→ [Adapter] ──→ In-Memory
                 └──────────────────────┘
```

### How It Works

**Ports** are interfaces (traits in Rust, interfaces in Java/Go) that define what the domain needs. There are two kinds:
- **Driving ports** (primary/inbound): How the outside world calls into the domain. Example: a `PlaceOrderUseCase` trait.
- **Driven ports** (secondary/outbound): What the domain needs from the outside world. Example: an `OrderRepository` trait.

**Adapters** are concrete implementations of ports:
- **Driving adapters**: HTTP handlers, CLI parsers, gRPC services, test harnesses.
- **Driven adapters**: PostgreSQL repositories, Redis caches, SMTP email senders, in-memory fakes for testing.

### Detailed Rust Example

```text
// === DOMAIN (center of the hexagon) ===

// Value objects
STRUCTURE Money:
    amount_cents ← integer
    currency ← Currency

// Entity / Aggregate root
STRUCTURE Order:
    id ← OrderId
    items ← list of OrderItem
    status ← OrderStatus

PROCEDURE NEW_ORDER(items):
    IF items IS empty THEN
        RETURN Error(EmptyOrder)
    RETURN Order {
        id ← GENERATE_ORDER_ID(),
        items ← items,
        status ← Pending
    }

// === PORTS (interfaces defined in the domain) ===

// Driven port: what the domain needs for persistence
INTERFACE OrderRepository:
    PROCEDURE SAVE(order) → Result
    PROCEDURE FIND_BY_ID(id) → optional Order

// Driven port: what the domain needs for notifications
INTERFACE OrderNotifier:
    PROCEDURE NOTIFY_ORDER_PLACED(order) → Result

// Driving port: the use case the domain exposes
INTERFACE PlaceOrderUseCase:
    PROCEDURE PLACE_ORDER(items) → Order or Error

// === APPLICATION SERVICE (implements driving port, uses driven ports) ===

STRUCTURE OrderService:
    repo ← OrderRepository
    notifier ← OrderNotifier

IMPLEMENT PlaceOrderUseCase FOR OrderService:
    PROCEDURE PLACE_ORDER(items):
        order ← NEW_ORDER(items)
        IF order IS error THEN RETURN error
        AWAIT self.repo.SAVE(order)
        AWAIT self.notifier.NOTIFY_ORDER_PLACED(order)
        RETURN order

// === ADAPTERS (outside the hexagon) ===

// Driven adapter: PostgreSQL implementation
STRUCTURE PgOrderRepository:
    pool ← PgPool

IMPLEMENT OrderRepository FOR PgOrderRepository:
    PROCEDURE SAVE(order):
        EXECUTE SQL "INSERT INTO orders ..." USING self.pool
        IF error THEN RETURN Error(Persistence, error message)
    // ...

// Driven adapter: In-memory implementation (for tests)
STRUCTURE InMemoryOrderRepository:
    orders ← shared map of OrderId → Order

// Driving adapter: HTTP handler
PROCEDURE PLACE_ORDER_HANDLER(service, request):
    order ← AWAIT service.PLACE_ORDER(request.items)
    IF order IS error THEN RETURN ApiError
    RETURN JSON(order)
```

### Real-World Examples

**Netflix's Zuul gateway** uses hexagonal architecture internally. The core routing logic is isolated from HTTP specifics, allowing them to test routing decisions without spinning up an HTTP server.

**Spotify's payment system** adopted ports and adapters to handle multiple payment providers (Stripe, PayPal, carrier billing). The domain defines a `PaymentGateway` port, and each provider is an adapter. Adding a new payment provider means writing one adapter with zero changes to business logic.

**Government digital services (UK GDS, US 18F)** frequently use hexagonal architecture because government projects often need to swap vendors. When a database vendor contract expires, only the adapter changes.

### Strengths

- **Testability**: The domain can be tested with in-memory adapters. No database, no network, no filesystem. Tests run in milliseconds.
- **Technology independence**: The domain does not import `sqlx`, `reqwest`, or `axum`. Swapping PostgreSQL for DynamoDB means writing a new adapter.
- **Multiple entry points**: The same domain logic can be accessed via HTTP, gRPC, CLI, or a message queue. Each is just a different driving adapter.
- **Enforced boundaries**: The compiler (especially in Rust with its trait system) enforces that the domain cannot accidentally depend on infrastructure.

### Weaknesses and Pitfalls

- **Over-engineering for simple CRUD**: If your application is "receive data, validate it, store it, return it," hexagonal architecture adds indirection without proportional benefit.
- **Trait explosion**: It is easy to end up with dozens of traits with single implementations. If a trait will only ever have one implementation in production, question whether the abstraction earns its keep.
- **Mapping fatigue**: Data often needs to be converted between types at each boundary: HTTP DTO to domain type to database row. This is tedious but deliberate -- each layer has its own data shape for good reason.

### When to Use

- Systems with complex business logic that must be tested independently of infrastructure
- Projects likely to change infrastructure (database, cloud provider, message broker)
- Applications serving multiple interfaces (web, mobile API, CLI, event consumers)
- Long-lived systems where technology choices will evolve over years

### Rust Project Structure Example

```
src/
├── main.rs                    # Wiring / composition root
├── domain/                    # The hexagon center
│   ├── mod.rs
│   ├── order.rs               # Entities, value objects, domain logic
│   ├── errors.rs              # Domain error types
│   └── ports/                 # Trait definitions
│       ├── mod.rs
│       ├── order_repository.rs
│       └── order_notifier.rs
├── application/               # Use case orchestration
│   ├── mod.rs
│   └── order_service.rs
└── adapters/                  # Outside the hexagon
    ├── inbound/
    │   ├── http/              # Axum handlers
    │   └── cli/               # CLI commands
    └── outbound/
        ├── postgres/          # PostgreSQL adapter
        ├── email/             # SMTP adapter
        └── in_memory/         # Test doubles
```

---

## Clean Architecture

### Concept

Robert C. Martin's (Uncle Bob's) 2012 formalization of ideas from hexagonal architecture, Onion Architecture (Jeffrey Palermo), and BCE (Ivar Jacobson). It organizes code into concentric layers with a strict rule: **source code dependencies must only point inward**.

```
┌───────────────────────────────────────────────┐
│              Frameworks & Drivers              │  Web, DB, External APIs
├───────────────────────────────────────────────┤
│              Interface Adapters                │  Controllers, Presenters, Gateways
├───────────────────────────────────────────────┤
│              Application Business Rules        │  Use Cases
├───────────────────────────────────────────────┤
│              Enterprise Business Rules         │  Entities
└───────────────────────────────────────────────┘

                Dependencies point INWARD
```

### The Dependency Rule

This is the single most important principle in clean architecture. Inner layers know nothing about outer layers:

- **Entities** (innermost): Pure business objects and rules. No framework imports, no database awareness. These could run on any platform.
- **Use Cases**: Application-specific business rules. Orchestrate entities and define the system's behavior. Know about entities but not about controllers or databases.
- **Interface Adapters**: Convert data between use case format and external format. Controllers parse HTTP, presenters format responses, gateways translate database rows.
- **Frameworks & Drivers**: The outermost layer. Web frameworks, database drivers, message queue clients. These are "details" that can be swapped.

### How It Differs from Hexagonal

Clean architecture and hexagonal architecture share the same core principle (dependency inversion toward the domain). The differences are:

| Aspect | Hexagonal | Clean |
|--------|-----------|-------|
| **Layers** | Two regions: inside and outside the hexagon | Four explicit concentric layers |
| **Use cases** | Implicit (part of the domain) | Explicit layer with its own responsibility |
| **Naming** | Ports and adapters | Entities, use cases, interface adapters, frameworks |
| **Prescriptiveness** | More flexible | More structured, with specific rules per layer |

In practice, teams often blend the two. The labels matter less than the principle: dependencies point inward.

### Real-World Examples

**Android development** widely adopted clean architecture after Google recommended it. A typical Android app has an `entities` module (Kotlin data classes with business rules), a `usecases` module (application logic), and `presentation`/`data` modules (UI and API/database respectively).

**Banking and fintech** (Nubank, Revolut) use clean architecture for regulatory compliance. Auditors can inspect the entities layer to verify business rules without wading through HTTP framework code.

**Game engines** often follow clean architecture principles even if they do not name it as such. The game logic (entities + rules) is isolated from rendering, input, and networking, which allows the same game to run on PC, console, and mobile with different adapter layers.

### Strengths

- Everything from hexagonal architecture (testability, technology independence) plus:
- **Explicit use cases**: The application's behavior is documented in code. Each use case is a class/struct with a clear purpose, making the system's capabilities discoverable.
- **Screaming architecture**: Looking at the use cases directory tells you what the system does (PlaceOrder, CancelOrder, GenerateInvoice) rather than what framework it uses.

### Weaknesses and Pitfalls

- **Ceremony for small projects**: Four layers with strict rules create significant boilerplate for a simple CRUD app.
- **Dogmatic application**: Some teams enforce the rules so rigidly that productivity suffers. Pragmatism matters. If your use case just calls a repository method and returns the result, maybe it does not need its own class.
- **Over-abstraction of entities**: Not every domain warrants rich entity behavior. Sometimes a struct with fields is enough.

### When to Use

- Medium-to-large systems with non-trivial business logic
- Teams that want explicit, discoverable use cases
- Projects where multiple developers work on different layers simultaneously
- Systems requiring long-term maintainability over years or decades

---

## Comparison Summary

| Criterion | Layered | Hexagonal | Clean |
|-----------|---------|-----------|-------|
| **Complexity** | Low | Medium | Medium-High |
| **Testability** | Moderate (needs DB mocks) | High (in-memory adapters) | High |
| **Flexibility** | Low (coupled to DB) | High | High |
| **Onboarding** | Fast | Moderate | Slower |
| **Best for** | CRUD apps, small teams | Complex domains, multi-interface | Large teams, long-lived systems |
| **Avoid for** | Complex domains | Simple CRUD | Small projects, prototypes |

### Event-Driven Architecture (Brief Note)

While not covered in depth here, event-driven architecture is often combined with the styles above. Components communicate by producing and consuming events rather than calling each other directly:

```
[Order Service] ──publishes──→ "OrderPlaced" ──→ [Message Broker]
                                                       │
                              ┌─────────────────────────┤
                              ↓                         ↓
                    [Inventory Service]          [Email Service]
```

Event-driven patterns are orthogonal to hexagonal/clean architecture. A hexagonal system can use events internally (between modules) or externally (between services). The event bus is simply another driven adapter.

---

## Further Reading

- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) -- Alistair Cockburn's original article
- *Clean Architecture* -- Robert C. Martin (2017)
- *Fundamentals of Software Architecture* -- Mark Richards & Neal Ford (2020)
- *Making Architecture Matter* -- Martin Fowler (OSCON talk)
