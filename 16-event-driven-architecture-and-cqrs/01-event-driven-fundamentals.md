# Event-Driven Fundamentals

## What Is Event-Driven Architecture?

In event-driven architecture (EDA), components communicate by producing and reacting to events rather than calling each other directly. An event represents a fact -- something that happened in the past.

```
Traditional (request/response):
  OrderService --calls--> InventoryService --calls--> EmailService

Event-driven:
  OrderService --publishes "OrderPlaced"--> [Event Bus]
                                                |
                        +-----------------------+
                        v                       v
                  InventoryService         EmailService
                  (reacts to event)        (reacts to event)
```

In the traditional model, OrderService must know about InventoryService and EmailService. In EDA, OrderService publishes an event and does not care who consumes it. This is *loose coupling* -- services can be added, removed, or changed without modifying the producer.

## Events vs Commands vs Queries

These three concepts look similar on the surface, but they differ in fundamental ways that shape system design.

| Type | Direction | Semantics | Example |
|------|-----------|-----------|---------|
| **Event** | Broadcast (one-to-many) | Something that happened (past tense) | `OrderPlaced`, `PaymentReceived`, `UserRegistered` |
| **Command** | Targeted (one-to-one) | A request to do something (imperative) | `PlaceOrder`, `ProcessPayment`, `SendEmail` |
| **Query** | Targeted (one-to-one) | A request for information | `GetOrderById`, `ListUserOrders` |

**Events are facts, not requests.** You cannot reject an event -- it already happened. You can only react to it. Commands can be accepted or rejected.

### Why This Distinction Matters

Confusing events and commands leads to coupling. If you name an event `UserShouldBeDeleted`, you have created a command disguised as an event. The producing service is now directing the consuming service, which defeats the purpose of EDA. Events describe what happened (`UserDeleted`), not what should happen.

```text
// WRONG: Command disguised as an event
STRUCTURE SendWelcomeEmail { user_id, email }

// RIGHT: Event -- consumers decide what to do
STRUCTURE UserRegistered { user_id, email, registered_at }
```

When `UserRegistered` is published, the email service can send a welcome email, the analytics service can update signup metrics, and a referral service can check for referral bonuses. The producer knows nothing about any of these reactions.

## Domain Events vs Integration Events

**Domain events** are internal to a bounded context. They represent something significant within the domain model.

```text
// Domain event -- internal to the Order context
ENUMERATION OrderDomainEvent:
    ItemAdded { order_id, item }
    DiscountApplied { order_id, discount }
    OrderSubmitted { order_id, total }
```

**Integration events** cross bounded context boundaries. They are the external-facing contract between services.

```text
// Integration event -- published to other services
STRUCTURE OrderPlacedEvent (serializable):
    event_id ← UUID
    order_id ← string
    customer_id ← string
    total_amount ← float
    currency ← string
    items ← list of OrderItemDto
    occurred_at ← datetime
```

**Why the distinction matters:** Domain events can change freely (internal implementation detail). Integration events are part of a public contract -- changing them can break other services. Treat integration events like API versions.

### Mapping Between the Two

A single domain event may produce zero or more integration events. An `OrderSubmitted` domain event might produce an `OrderPlaced` integration event for external consumers, while `DiscountApplied` stays internal because no other service needs to know about discounting logic.

```text
PROCEDURE TO_INTEGRATION_EVENT(domain_event):
    MATCH domain_event:
        OrderSubmitted { order_id, total } →
            RETURN IntegrationEvent.OrderPlaced {
                event_id ← NEW_UUID(),
                order_id ← TO_STRING(order_id),
                total_amount ← total AS float,
                occurred_at ← NOW()
            }
        // Internal events -- not published externally
        _ → RETURN None
```

## Event-Driven vs Request/Response

Request/response is synchronous and tightly coupled: the caller blocks until the callee responds, and both must be online. Event-driven communication is asynchronous and loosely coupled: the producer publishes and moves on.

| Property | Request/Response | Event-Driven |
|----------|-----------------|--------------|
| Coupling | Tight -- caller knows callee | Loose -- producer does not know consumers |
| Availability | Both must be online | Producer and consumer can be independently available |
| Scaling | Scale together | Scale independently |
| Debugging | Easy to trace call chain | Harder to trace event flows |
| Adding consumers | Modify the caller | Subscribe to existing events |

Neither approach is universally better. Use request/response when you need an immediate answer (user login, payment authorization). Use events when you need loose coupling and asynchronous reactions (notifications, analytics, inventory updates).

## Real-World Examples

### Walmart's Event-Driven Inventory

Walmart's inventory system uses EDA to track billions of inventory changes across 11,000+ stores. When an item is sold, an event is published. Multiple consumers react: inventory counts update, reorder alerts trigger, analytics update, and regional warehouses adjust. Trying to manage this with synchronous calls would be impossibly complex and fragile.

The key insight is fan-out: one event triggers many independent reactions. Adding a new consumer (e.g., a fraud detection system that monitors unusual purchase patterns) requires zero changes to the point-of-sale system that produces the event.

### Uber's Event-Driven Dispatch

Uber's ride-matching system is event-driven. Events include: ride requested, driver location updated, ride accepted, trip started, trip completed. Each event triggers downstream processes (ETA calculation, pricing, notifications). The event-driven design enabled Uber to add new features (tipping, ratings, surge pricing) by adding new event consumers without modifying the core dispatch system.

Before their event-driven redesign, adding a feature like real-time surge pricing would have required changes to the dispatch service. With EDA, a new surge-pricing service subscribes to ride-requested and driver-location events and publishes pricing adjustments independently.

## Common Mistakes

- **Overly fine-grained events** -- An event for every field change (`UserEmailChanged`, `UserNameChanged`, `UserPhoneChanged`). Group related changes into meaningful domain events (`UserProfileUpdated`).

- **Not handling duplicate events** -- Events may be delivered more than once (at-least-once delivery). Consumers must be idempotent -- processing the same event twice should produce no additional effect.

- **Ignoring event ordering** -- Events for the same entity should be processed in order. Kafka guarantees ordering within a partition. Use the entity ID as the partition key.

- **Event schema evolution** -- Events are stored forever. Adding a new field is easy (use defaults). Removing or renaming fields breaks old events. Use versioning or schema registries.

## Key Takeaways

1. Events represent facts -- things that happened. They are immutable and form an append-only log.
2. Commands request action; events report what already happened. Never confuse the two.
3. Domain events are internal; integration events are external contracts. Treat them differently.
4. EDA shines when you need loose coupling and fan-out. Use request/response when you need immediate answers.
5. Event consumers must be idempotent. Assume every event will be delivered at least once.
