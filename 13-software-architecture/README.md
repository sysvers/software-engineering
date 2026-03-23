# 13 - Software Architecture

## Diagrams

![Bounded Contexts](diagrams/bounded-contexts.png)

![Hexagonal Architecture](diagrams/hexagonal-architecture.png)

## Contents

1. [Architectural Styles](01-architectural-styles.md) — Layered, Hexagonal (Ports & Adapters), and Clean Architecture. How each works, when to use them, Rust project structures, and real-world examples.

2. [Domain-Driven Design](02-domain-driven-design.md) — Bounded Contexts, Aggregates, Entities vs. Value Objects, Domain Events, Context Mapping, and the Anti-Corruption Layer pattern.

3. [Modular Monolith](03-modular-monolith.md) — Why start with a monolith, module boundaries, Rust visibility enforcement with Cargo workspaces, synchronous vs. asynchronous inter-module communication, and the path to microservices.

4. [Architecture Decision Records](04-architecture-decision-records.md) — What ADRs are, the standard template (Context, Decision, Alternatives, Consequences), sample ADRs, writing guidelines, adr-tools, storage conventions, and common mistakes.

5. [Dependency Inversion and Separation of Concerns](05-dependency-inversion-and-separation.md) — Separation of concerns at the architecture level, the Dependency Inversion Principle, dependency injection in Rust using traits and trait objects, the composition root pattern, Conway's Law, and Amazon's API mandate.

## Key Takeaways

- Architecture is the set of decisions that are expensive to change. Invest in getting them right, but do not over-invest.
- Start with a modular monolith. Extract microservices only when you have a concrete scaling or organizational reason.
- The Dependency Rule: dependencies point inward, toward the domain. The domain never depends on infrastructure.
- Bounded Contexts prevent the God Object problem. The same word can mean different things in different parts of the system.
- Conway's Law is unavoidable. Design your teams to match the architecture you want.
- Write ADRs. Fifteen minutes of writing saves months of archaeology.
