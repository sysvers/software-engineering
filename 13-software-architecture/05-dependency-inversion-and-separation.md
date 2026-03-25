# Dependency Inversion and Separation of Concerns

## Overview

Two principles sit at the heart of every well-architected system: **Separation of Concerns** (each component has one job) and **Dependency Inversion** (high-level policy does not depend on low-level detail). Together, they determine whether a codebase remains maintainable as it grows or collapses under its own weight.

This document covers both principles at the architecture level, explores how Rust's trait system enables dependency injection without a framework, and examines Conway's Law — the organizational force that shapes (and misshapes) architecture whether teams acknowledge it or not.

---

## Separation of Concerns at the Architecture Level

### The Principle

Every module, layer, or service should have a single reason to change. This is the Single Responsibility Principle applied not to individual functions but to entire subsystems.

When concerns are separated, a change to how emails are sent does not require modifying the order processing logic. A change to the database schema does not ripple into the HTTP handler. Each concern is isolated behind a boundary.

### What Counts as a "Concern"

At the architecture level, concerns are coarser than at the code level:

- **Business logic**: The rules that define what the system does (pricing, validation, workflows)
- **Persistence**: How and where data is stored (PostgreSQL, Redis, filesystem)
- **Transport**: How the system communicates with the outside world (HTTP, gRPC, CLI, message queues)
- **Authentication/Authorization**: Who is allowed to do what
- **Observability**: Logging, metrics, tracing
- **External integrations**: Third-party APIs, payment providers, email services

Each of these concerns should be isolable. You should be able to swap PostgreSQL for DynamoDB without touching business logic. You should be able to replace HTTP with gRPC without modifying persistence.

### The Violation Pattern

The most common violation is the "God handler" — an HTTP endpoint that mixes transport, auth, business logic, persistence, and external integrations in a single function. It has six reasons to change, and testing it requires every piece of infrastructure to be running.

### The Separated Pattern

```text
// Transport layer: only HTTP concerns
PROCEDURE CREATE_ORDER_HANDLER(service, user, request):
    order ← AWAIT service.CREATE_ORDER(user.id, request)
    IF order IS error THEN RETURN ApiError
    RETURN JSON(order)

// Application layer: orchestrates the use case
PROCEDURE CREATE_ORDER(service, user_id, input):
    CHECK_PERMISSION(user_id, CreateOrder)
    order ← NEW_ORDER(user_id, input.items)
    IF order IS error THEN RETURN error
    AWAIT service.repo.SAVE(order)
    AWAIT service.events.PUBLISH(OrderCreated FROM order)
    RETURN order

// Domain layer: pure business rules, no imports from infrastructure
PROCEDURE NEW_ORDER(user_id, items):
    IF items IS empty THEN RETURN Error(EmptyOrder)
    total ← SUM OF LINE_TOTAL(item) FOR EACH item IN items
    RETURN Order { id ← GENERATE_ID(), user_id, items, total, status ← Pending }
```

Each layer has one concern. The HTTP handler knows nothing about databases. The domain knows nothing about HTTP. Testing the domain requires no infrastructure at all.

---

## The Dependency Inversion Principle

### The Problem

In a naive architecture, high-level modules depend on low-level modules:

```
OrderService → PostgresRepository → sqlx → libpq → PostgreSQL
```

The business logic (OrderService) depends on a specific database technology. To test it, you need PostgreSQL. To switch databases, you modify business logic. The most important code (business rules) is coupled to the most volatile code (infrastructure details).

### The Inversion

The Dependency Inversion Principle (the "D" in SOLID) states:

1. High-level modules should not depend on low-level modules. Both should depend on abstractions.
2. Abstractions should not depend on details. Details should depend on abstractions.

After inversion:

```
OrderService → OrderRepository (trait)    ← defined in the domain
                     ↑
         PostgresOrderRepo (implements trait)  ← defined in infrastructure
```

The arrow between `PostgresOrderRepo` and the trait points upward — the infrastructure depends on the domain's abstraction, not the other way around. This is the "inversion": the dependency direction has been reversed.

### Dependency Injection in Rust Using Traits

Rust does not have a DI framework like Spring (Java) or .NET's built-in container. It does not need one. Rust's trait system and generics provide compile-time dependency injection that is more explicit and carries zero runtime cost.

#### Generic-Based Injection

```text
// The abstraction: defined in the domain layer
INTERFACE OrderRepository:
    PROCEDURE SAVE(order) → Result
    PROCEDURE FIND_BY_ID(id) → optional Order or Error

// The high-level module: depends on the abstraction, not a concrete type
STRUCTURE OrderService:
    repo ← OrderRepository

PROCEDURE NEW_ORDER_SERVICE(repo):
    RETURN OrderService { repo ← repo }

PROCEDURE GET_ORDER(service, id):
    result ← AWAIT service.repo.FIND_BY_ID(id)
    IF result IS NOT found THEN RETURN Error(NotFound)
    RETURN result

// Production: uses PostgreSQL
repo ← NEW PostgresOrderRepo(pool)
service ← NEW_ORDER_SERVICE(repo)

// Test: uses in-memory fake
repo ← NEW InMemoryOrderRepo()
service ← NEW_ORDER_SERVICE(repo)
```

The compiler monomorphizes `OrderService<PostgresOrderRepo>` and `OrderService<InMemoryOrderRepo>` into separate, specialized types. There is no vtable, no dynamic dispatch, no runtime cost.

#### Trait Object-Based Injection

When you need to decide the implementation at runtime (e.g., based on configuration) or want to avoid generic proliferation, use trait objects:

```text
STRUCTURE OrderService:
    repo ← shared reference to OrderRepository (dynamic dispatch)

PROCEDURE NEW_ORDER_SERVICE(repo):
    RETURN OrderService { repo ← repo }

// Wiring at the composition root (main.rs)
IF config.use_mock_db THEN
    repo ← NEW InMemoryOrderRepo()
ELSE
    repo ← NEW PostgresOrderRepo(pool)
service ← NEW_ORDER_SERVICE(repo)
```

This incurs a small runtime cost (dynamic dispatch through a vtable), which is negligible for I/O-bound operations like database access.

#### The Composition Root

All the wiring — connecting abstractions to implementations — happens in one place: `main.rs` (the composition root). This is the only place in the codebase that knows about concrete types. It creates the implementations, connects them to the abstractions, and starts the application. Every other module works exclusively with traits.

---

## Conway's Law

### The Law

> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations." — Melvin Conway, 1967

This is not a suggestion. It is an observation about an unavoidable force. If your organization has a frontend team, a backend team, and a database team, you will get a three-tier architecture — regardless of what architecture you intended.

### The Inverse Conway Maneuver

If Conway's Law is unavoidable, use it deliberately. Structure your teams to match the architecture you want. If you want a modular monolith with Orders, Inventory, and Payments modules, create three domain teams each owning one module. If you want microservices, create small autonomous teams each owning one or more services. If you want loosely coupled services, you need loosely coupled teams.

### Real-World Example: Amazon's API Mandate

In 2002, Jeff Bezos issued what became known as the "Bezos API Mandate" — an internal memo that changed how Amazon built software:

1. All teams will expose their data and functionality through service interfaces.
2. Teams must communicate with each other through these interfaces.
3. There will be no other form of inter-process communication allowed: no direct linking, no direct reads of another team's data store, no shared-memory model, no back-doors.
4. It does not matter what technology they use.
5. All service interfaces, without exception, must be designed from the ground up to be externalizable.
6. Anyone who does not do this will be fired.

The mandate forced architectural separation by mandating organizational separation. Teams could no longer share databases or call internal functions across team boundaries. Every interaction required a well-defined API.

The consequences were profound. Because every internal service had an externalizable API, Amazon realized they could sell those services externally — S3, SQS, and EC2 started as internal infrastructure. Teams gained full autonomy over technology choices; the only contract was the API. Without shared databases, teams could not take shortcuts, and data ownership became clear. This is Conway's Law applied deliberately: the organizational mandate produced the desired architecture.

---

## The Interplay: Separation, Inversion, and Organization

These three concepts reinforce each other:

1. **Separation of Concerns** tells you what to separate (business logic from infrastructure, modules from each other).
2. **Dependency Inversion** tells you how to separate it (abstractions owned by the domain, implementations in infrastructure).
3. **Conway's Law** tells you who separates it (team structure must match the desired boundaries).

A system where all three align — clear concerns, inverted dependencies, and matching team structure — is one that can evolve for years without accumulating crippling technical debt. A system where any one is missing will struggle: perfect dependency inversion in a badly organized team produces clean code that nobody owns; perfect team structure with tangled dependencies produces autonomous teams blocked by shared code.

---

## Common Mistakes

### Abstracting too early
Creating traits for every dependency "just in case" leads to trait explosion. Abstract when you have a concrete reason: testability, multiple implementations, or a boundary you need to enforce. A function that will only ever call one database is not improved by wrapping it in a trait.

### Circular dependencies between modules
If module A depends on module B and module B depends on module A, the boundary is in the wrong place. Extract the shared concept into a third module, or use events to break the cycle.

### Ignoring Conway's Law
Designing a beautiful microservice architecture while organized as a single team with shared code ownership produces a distributed monolith — microservices that must be deployed together because they are developed together.

### Dependency inversion without a composition root
Scattering the wiring (which concrete type implements which trait) across the codebase defeats the purpose. All wiring belongs in one place — `main.rs` or a dedicated module — so that the full dependency graph is visible in one location.

### Confusing dependency inversion with dependency injection
Dependency injection is a technique (passing dependencies in from outside). Dependency inversion is a principle (high-level modules define abstractions that low-level modules implement). You can have injection without inversion (passing a concrete `PostgresRepo` into a service), and you can have inversion without injection (using a static factory). The principle matters more than the technique.

---

## Further Reading

- *Clean Architecture* -- Robert C. Martin (2017) -- Chapters on the Dependency Inversion Principle and the Dependency Rule
- *Domain-Driven Design* -- Eric Evans (2003) -- Bounded contexts as organizational and architectural boundaries
- [Conway's Law](https://www.melconway.com/Home/Conways_Law.html) -- Melvin Conway's original paper
- [The Bezos API Mandate](https://gist.github.com/chitchcock/1281611) -- Steve Yegge's account of Amazon's organizational transformation
- *Team Topologies* -- Matthew Skelton & Manuel Pais (2019) -- Applying Conway's Law deliberately through team design
- *Fundamentals of Software Architecture* -- Mark Richards & Neal Ford (2020) -- Comprehensive coverage of architectural principles
