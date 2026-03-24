# Event-Driven Architecture & CQRS

## Contents

1. [Event-Driven Fundamentals](01-event-driven-fundamentals.md) -- Events vs commands, domain vs integration events, event buses, loose coupling
2. [Event Sourcing](02-event-sourcing.md) -- Append-only event stores, rebuilding state from events, optimistic concurrency
3. [Projections and Snapshots](03-projections-and-snapshots.md) -- Building read models from events, snapshot strategies, projection rebuilds
4. [CQRS](04-cqrs.md) -- Separating read and write models, eventual consistency, when CQRS is overkill
5. [Sagas and Compensation](05-sagas-and-compensation.md) -- Choreography vs orchestration, saga pattern, compensating actions, distributed transactions (2PC), failure scenarios

## Diagrams

![CQRS Pattern](diagrams/cqrs-pattern.png)

![Saga Pattern](diagrams/saga-pattern.png)
