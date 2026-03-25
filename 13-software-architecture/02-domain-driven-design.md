# Domain-Driven Design (DDD)

## Overview

Domain-Driven Design is an approach to software development that centers the architecture and codebase around the business domain. Created by Eric Evans in his 2003 book *Domain-Driven Design: Tackling Complexity in the Heart of Software*, DDD provides patterns for modeling complex systems so that the code mirrors the language and structure of the business it serves.

DDD is not a framework or a technology choice. It is a set of principles for how to think about software structure when the primary source of complexity is the business domain itself, not the infrastructure.

---

## When DDD Applies (and When It Does Not)

DDD is valuable when:
- The domain is complex and requires deep understanding (insurance underwriting, supply chain logistics, financial trading)
- Business rules are the primary source of complexity
- You have access to domain experts who can collaborate with developers
- The system will evolve over years and the domain model needs to stay aligned with the business

DDD is overkill when:
- The application is primarily CRUD with thin business logic
- The domain is well-understood and stable (a blog, a static site generator)
- The team is small and the entire domain fits in one person's head

---

## Bounded Contexts

### Concept

A bounded context is the most important strategic pattern in DDD. It defines an explicit boundary within which a domain model exists. Inside the boundary, every term has a precise, unambiguous meaning. Outside the boundary, the same term might mean something completely different.

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Sales Context   │    │  Shipping Context│    │  Billing Context│
│                  │    │                  │    │                  │
│  Customer:       │    │  Customer:       │    │  Customer:       │
│  - name          │    │  - address       │    │  - payment method│
│  - preferences   │    │  - delivery zone │    │  - credit limit  │
│  - purchase hist │    │  - contact phone │    │  - billing addr  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

"Customer" means something different in each context. In Sales, a customer has preferences and purchase history. In Shipping, a customer has an address and delivery zone. In Billing, a customer has payment methods and credit limits.

### The God Object Anti-Pattern

Without bounded contexts, teams inevitably create one massive `Customer` struct/class that tries to serve every part of the system:

```text
// The God Object -- DO NOT DO THIS
STRUCTURE Customer:
    id ← CustomerId
    name ← string
    email ← string
    preferences ← list of Preference       // Sales needs this
    purchase_history ← list of Purchase     // Sales needs this
    shipping_address ← Address              // Shipping needs this
    delivery_zone ← DeliveryZone            // Shipping needs this
    contact_phone ← string                  // Shipping needs this
    payment_method ← PaymentMethod          // Billing needs this
    credit_limit ← Money                    // Billing needs this
    billing_address ← Address               // Billing needs this
    tax_id ← optional string                // Billing needs this
    loyalty_points ← integer                // Marketing needs this
    segment ← CustomerSegment               // Marketing needs this
```

This object grows to dozens of fields. Every change risks breaking unrelated features. Loading a Customer for shipping pulls in billing data it does not need. The struct becomes a coordination bottleneck across teams.

### Correct Approach: Separate Models Per Context

```text
// Sales context
MODULE sales:
    STRUCTURE Customer:
        id ← CustomerId
        name ← string
        preferences ← list of Preference
        purchase_history ← list of PurchaseId

// Shipping context
MODULE shipping:
    STRUCTURE Recipient:
        customer_id ← CustomerId  // Reference to sales context
        address ← Address
        delivery_zone ← DeliveryZone
        contact_phone ← string

// Billing context
MODULE billing:
    STRUCTURE BillingAccount:
        customer_id ← CustomerId  // Reference to sales context
        payment_method ← PaymentMethod
        credit_limit ← Money
        billing_address ← Address
```

Each context has its own model with only the data it needs. They share a `CustomerId` to correlate across contexts, but each model is independently owned and evolved.

### Context Mapping

Bounded contexts do not exist in isolation. They interact. DDD defines several relationship patterns:

- **Shared Kernel**: Two contexts share a small, jointly-owned model. Both teams must agree on changes. Use sparingly.
- **Customer-Supplier**: One context (supplier) provides data to another (customer). The supplier team may or may not accommodate the customer's needs.
- **Anti-Corruption Layer (ACL)**: A translation layer that prevents one context's model from leaking into another. Critical when integrating with legacy systems or third-party APIs.
- **Published Language**: A well-documented, shared language (often a schema or API contract) that contexts use to communicate.

### Real-World Example: Anti-Corruption Layer

```text
// External payment provider returns this
STRUCTURE StripeCharge:
    id ← string
    amount ← integer          // Stripe uses cents
    currency ← string         // Stripe uses ISO strings
    status ← string           // "succeeded", "failed", etc.

// Our domain model
STRUCTURE Payment:
    id ← PaymentId
    amount ← Money
    status ← PaymentStatus

// Anti-corruption layer: translates Stripe's model into ours
PROCEDURE CONVERT_STRIPE_CHARGE_TO_PAYMENT(charge):
    id ← PaymentId.FROM_EXTERNAL(charge.id)
    amount ← Money.FROM_CENTS(charge.amount, PARSE_CURRENCY(charge.currency))

    MATCH charge.status:
        "succeeded" → status ← Completed
        "failed"    → status ← Failed
        "pending"   → status ← Processing
        other       → RETURN Error(UnknownPaymentStatus, other)

    RETURN Payment { id, amount, status }
```

The ACL ensures that if Stripe changes their API (renames fields, changes status strings), only the translation layer changes. The domain model remains stable.

---

## Aggregates

### Concept

An aggregate is a cluster of domain objects that are treated as a single unit for the purpose of data changes. Every aggregate has a **root entity** (the aggregate root), and all external access to the aggregate goes through the root.

The aggregate root enforces **invariants**: business rules that must always be true. For example, "an order cannot have a negative total" or "an order with status Shipped cannot be modified."

### Rules for Aggregates

1. **External references only to the root**: Other parts of the system hold references to the aggregate root (by ID), never to internal entities.
2. **Transactional boundary**: An aggregate is saved/loaded as a unit. One database transaction per aggregate.
3. **Consistency within, eventual consistency between**: Within an aggregate, all invariants are immediately enforced. Between aggregates, eventual consistency is acceptable.
4. **Keep them small**: Large aggregates create contention (multiple users trying to modify the same aggregate simultaneously). Prefer smaller aggregates with references between them.

### Rust Example

```text
STRUCTURE Order:
    id ← OrderId
    customer_id ← CustomerId
    items ← list of OrderItem
    status ← OrderStatus
    total ← Money
    created_at ← DateTime

// OrderItem is internal to the aggregate -- no external access
STRUCTURE OrderItem:
    product_id ← ProductId
    quantity ← integer
    unit_price ← Money
    line_total ← Money

PROCEDURE NEW_ORDER(customer_id):
    RETURN Order {
        id ← GENERATE_ORDER_ID(),
        customer_id ← customer_id,
        items ← empty list,
        status ← Draft,
        total ← Money.ZERO(USD),
        created_at ← NOW()
    }

// All mutations go through the aggregate root
PROCEDURE ADD_ITEM(order, product_id, quantity, unit_price):
    // Invariant: cannot modify submitted orders
    IF order.status ≠ Draft THEN
        RETURN Error(CannotModifySubmittedOrder)
    // Invariant: quantity must be positive
    IF quantity = 0 THEN
        RETURN Error(InvalidQuantity)

    line_total ← unit_price * quantity
    APPEND TO order.items: OrderItem { product_id, quantity, unit_price, line_total }
    RECALCULATE_TOTAL(order)
    RETURN Ok

PROCEDURE SUBMIT(order):
    // Invariant: cannot submit empty order
    IF order.items IS empty THEN
        RETURN Error(EmptyOrder)
    // Invariant: cannot submit twice
    IF order.status ≠ Draft THEN
        RETURN Error(AlreadySubmitted)

    order.status ← Submitted

    // Return domain events for other contexts to react to
    RETURN [OrderPlaced { order_id, customer_id, total }]

PROCEDURE RECALCULATE_TOTAL(order):
    order.total ← SUM OF item.line_total FOR EACH item IN order.items
```

### Common Mistake: Over-Sized Aggregates

A frequent error is making aggregates too large. For example, putting all `OrderItem` inventory checks inside the `Order` aggregate. This means modifying any item locks the entire order for the transaction.

Instead, keep aggregates small and reference other aggregates by ID:

```text
// WRONG: Product is part of Order aggregate
STRUCTURE Order:
    items ← list of Product  // Product is a full entity with its own lifecycle

// RIGHT: Order references products by ID
STRUCTURE Order:
    items ← list of OrderItem  // OrderItem holds ProductId, not Product
```

---

## Entities vs. Value Objects

### Entities

Entities have **identity**. Two entities with identical field values but different IDs are different objects. An entity's identity persists across time and state changes.

```text
STRUCTURE User:
    id ← UserId          // Identity -- this is what makes it unique
    name ← string
    email ← string

// These are DIFFERENT users, even with the same name and email
alice1 ← User { id ← UserId(1), name ← "Alice", email ← "alice@test.com" }
alice2 ← User { id ← UserId(2), name ← "Alice", email ← "alice@test.com" }
ASSERT alice1.id ≠ alice2.id  // Different entities
```

### Value Objects

Value objects have no identity. They are defined entirely by their attributes. Two value objects with the same attributes are interchangeable.

```text
STRUCTURE Money (value object, immutable, comparable by value):
    amount_cents ← integer
    currency ← Currency

PROCEDURE NEW_MONEY(amount_cents, currency):
    // Value objects validate on creation
    IF amount_cents < 0 THEN
        RETURN Error(NegativeAmount)
    RETURN Money { amount_cents, currency }

// Value objects are immutable -- operations return new instances
PROCEDURE ADD(self, other):
    IF self.currency ≠ other.currency THEN
        RETURN Error(CurrencyMismatch)
    RETURN NEW_MONEY(self.amount_cents + other.amount_cents, self.currency)
```

### Design Heuristics

- If you need to track the thing over time and distinguish between instances, it is an entity.
- If you care only about the attributes and could swap one instance for another with the same values, it is a value object.
- **Prefer value objects** when possible. They are simpler, immutable, and eliminate a class of bugs (identity confusion, aliasing problems).
- Common value objects: Money, EmailAddress, PhoneNumber, DateRange, Coordinates, Color.
- Common entities: User, Order, Product, Account, Invoice.

---

## Domain Events

### Concept

Domain events represent something significant that happened in the domain. They are named in past tense and describe facts that cannot be changed.

```text
ENUMERATION OrderEvent:
    OrderPlaced:
        order_id ← OrderId
        customer_id ← CustomerId
        total ← Money
        placed_at ← DateTime
    OrderShipped:
        order_id ← OrderId
        tracking_number ← string
        shipped_at ← DateTime
    OrderCancelled:
        order_id ← OrderId
        reason ← CancellationReason
        cancelled_at ← DateTime
```

### Why Domain Events Matter

Domain events decouple parts of the system. When an order is placed:
- The **Inventory** context reacts by reserving stock
- The **Email** context sends a confirmation
- The **Analytics** context records the sale
- The **Loyalty** context awards points

The Order context knows none of this. It publishes an event and moves on. New reactions can be added without modifying the Order context.

### Implementation Pattern in Rust

```text
// Aggregate collects events during operations
PROCEDURE SUBMIT(order):
    // ... validation ...
    order.status ← Submitted
    RETURN [OrderPlaced {
        order_id ← order.id,
        customer_id ← order.customer_id,
        total ← order.total,
        placed_at ← NOW()
    }]

// Application service publishes events after persisting
PROCEDURE SUBMIT_ORDER(service, order_id):
    order ← AWAIT service.repo.FIND_BY_ID(order_id)
    IF order IS NOT found THEN RETURN Error(NotFound)

    events ← SUBMIT(order)
    IF events IS error THEN RETURN error
    AWAIT service.repo.SAVE(order)

    // Publish after successful persistence
    FOR EACH event IN events:
        AWAIT service.event_publisher.PUBLISH(event)
    RETURN Ok
```

### Pitfalls with Domain Events

- **Dual-write problem**: Saving to the database and publishing to a message broker are two separate operations. If the publish fails after the save, the event is lost. Solutions include the Outbox pattern (save events to a database table, then relay them) or event sourcing.
- **Event ordering**: In distributed systems, events may arrive out of order. Consumers must handle this gracefully.
- **Event versioning**: As the domain evolves, event schemas change. Old events in the log need to remain readable. Use schema evolution strategies (adding optional fields, never removing fields).

---

## Real-World DDD Adoption

### Nubank (Brazilian fintech)

Nubank built Latin America's largest digital bank using DDD with Clojure. Their bounded contexts map to financial products: credit cards, savings accounts, personal loans, insurance. Each context has its own data store and domain model. Cross-context communication happens through events. This architecture allowed them to scale from one product (credit cards) to dozens while keeping teams autonomous.

### Shopify

Shopify's modular monolith is organized around bounded contexts: Orders, Products, Inventory, Payments, Shipping. Each context has clear ownership and interfaces. When a developer changes the Inventory context, they know the boundary and can verify they have not leaked implementation details to other contexts.

### UK Government (HMRC)

Her Majesty's Revenue and Customs rebuilt their tax platform using DDD. Tax domains (income tax, VAT, corporate tax) became bounded contexts. The ubiquitous language of each tax domain was developed with tax policy experts, ensuring the code used the same terminology as the legislation it implemented.

---

## Further Reading

- *Domain-Driven Design* -- Eric Evans (2003) -- The foundational text
- *Implementing Domain-Driven Design* -- Vaughn Vernon (2013) -- Practical patterns and examples
- *Domain Modeling Made Functional* -- Scott Wlaschin (2018) -- DDD with functional programming (many concepts transfer to Rust)
- *Learning Domain-Driven Design* -- Vlad Khononov (2021) -- A modern, accessible introduction
