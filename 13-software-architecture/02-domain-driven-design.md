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

```rust
// The God Object -- DO NOT DO THIS
struct Customer {
    id: CustomerId,
    name: String,
    email: String,
    preferences: Vec<Preference>,       // Sales needs this
    purchase_history: Vec<Purchase>,     // Sales needs this
    shipping_address: Address,           // Shipping needs this
    delivery_zone: DeliveryZone,         // Shipping needs this
    contact_phone: String,              // Shipping needs this
    payment_method: PaymentMethod,       // Billing needs this
    credit_limit: Money,                // Billing needs this
    billing_address: Address,           // Billing needs this
    tax_id: Option<String>,             // Billing needs this
    loyalty_points: u32,                // Marketing needs this
    segment: CustomerSegment,           // Marketing needs this
}
```

This object grows to dozens of fields. Every change risks breaking unrelated features. Loading a Customer for shipping pulls in billing data it does not need. The struct becomes a coordination bottleneck across teams.

### Correct Approach: Separate Models Per Context

```rust
// Sales context
mod sales {
    pub struct Customer {
        pub id: CustomerId,
        pub name: String,
        pub preferences: Vec<Preference>,
        pub purchase_history: Vec<PurchaseId>,
    }
}

// Shipping context
mod shipping {
    pub struct Recipient {
        pub customer_id: CustomerId,  // Reference to sales context
        pub address: Address,
        pub delivery_zone: DeliveryZone,
        pub contact_phone: String,
    }
}

// Billing context
mod billing {
    pub struct BillingAccount {
        pub customer_id: CustomerId,  // Reference to sales context
        pub payment_method: PaymentMethod,
        pub credit_limit: Money,
        pub billing_address: Address,
    }
}
```

Each context has its own model with only the data it needs. They share a `CustomerId` to correlate across contexts, but each model is independently owned and evolved.

### Context Mapping

Bounded contexts do not exist in isolation. They interact. DDD defines several relationship patterns:

- **Shared Kernel**: Two contexts share a small, jointly-owned model. Both teams must agree on changes. Use sparingly.
- **Customer-Supplier**: One context (supplier) provides data to another (customer). The supplier team may or may not accommodate the customer's needs.
- **Anti-Corruption Layer (ACL)**: A translation layer that prevents one context's model from leaking into another. Critical when integrating with legacy systems or third-party APIs.
- **Published Language**: A well-documented, shared language (often a schema or API contract) that contexts use to communicate.

### Real-World Example: Anti-Corruption Layer

```rust
// External payment provider returns this
#[derive(Deserialize)]
struct StripeCharge {
    id: String,
    amount: i64,          // Stripe uses cents
    currency: String,     // Stripe uses ISO strings
    status: String,       // "succeeded", "failed", etc.
}

// Our domain model
pub struct Payment {
    pub id: PaymentId,
    pub amount: Money,
    pub status: PaymentStatus,
}

// Anti-corruption layer: translates Stripe's model into ours
impl TryFrom<StripeCharge> for Payment {
    type Error = DomainError;

    fn try_from(charge: StripeCharge) -> Result<Self, Self::Error> {
        Ok(Payment {
            id: PaymentId::from_external(charge.id),
            amount: Money::from_cents(charge.amount, Currency::parse(&charge.currency)?),
            status: match charge.status.as_str() {
                "succeeded" => PaymentStatus::Completed,
                "failed" => PaymentStatus::Failed,
                "pending" => PaymentStatus::Processing,
                other => return Err(DomainError::UnknownPaymentStatus(other.to_string())),
            },
        })
    }
}
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

```rust
pub struct Order {
    id: OrderId,
    customer_id: CustomerId,
    items: Vec<OrderItem>,
    status: OrderStatus,
    total: Money,
    created_at: DateTime<Utc>,
}

// OrderItem is internal to the aggregate -- no external access
struct OrderItem {
    product_id: ProductId,
    quantity: u32,
    unit_price: Money,
    line_total: Money,
}

impl Order {
    pub fn new(customer_id: CustomerId) -> Self {
        Self {
            id: OrderId::generate(),
            customer_id,
            items: Vec::new(),
            status: OrderStatus::Draft,
            total: Money::zero(Currency::USD),
            created_at: Utc::now(),
        }
    }

    // All mutations go through the aggregate root
    pub fn add_item(
        &mut self,
        product_id: ProductId,
        quantity: u32,
        unit_price: Money,
    ) -> Result<(), OrderError> {
        // Invariant: cannot modify submitted orders
        if self.status != OrderStatus::Draft {
            return Err(OrderError::CannotModifySubmittedOrder);
        }
        // Invariant: quantity must be positive
        if quantity == 0 {
            return Err(OrderError::InvalidQuantity);
        }

        let line_total = unit_price.multiply(quantity);
        self.items.push(OrderItem {
            product_id,
            quantity,
            unit_price,
            line_total,
        });
        self.recalculate_total();
        Ok(())
    }

    pub fn submit(&mut self) -> Result<Vec<OrderEvent>, OrderError> {
        // Invariant: cannot submit empty order
        if self.items.is_empty() {
            return Err(OrderError::EmptyOrder);
        }
        // Invariant: cannot submit twice
        if self.status != OrderStatus::Draft {
            return Err(OrderError::AlreadySubmitted);
        }

        self.status = OrderStatus::Submitted;

        // Return domain events for other contexts to react to
        Ok(vec![OrderEvent::OrderPlaced {
            order_id: self.id.clone(),
            customer_id: self.customer_id.clone(),
            total: self.total.clone(),
        }])
    }

    fn recalculate_total(&mut self) {
        self.total = self.items.iter()
            .map(|item| &item.line_total)
            .fold(Money::zero(Currency::USD), |acc, m| acc.add(m));
    }
}
```

### Common Mistake: Over-Sized Aggregates

A frequent error is making aggregates too large. For example, putting all `OrderItem` inventory checks inside the `Order` aggregate. This means modifying any item locks the entire order for the transaction.

Instead, keep aggregates small and reference other aggregates by ID:

```rust
// WRONG: Product is part of Order aggregate
struct Order {
    items: Vec<Product>,  // Product is a full entity with its own lifecycle
}

// RIGHT: Order references products by ID
struct Order {
    items: Vec<OrderItem>,  // OrderItem holds ProductId, not Product
}
```

---

## Entities vs. Value Objects

### Entities

Entities have **identity**. Two entities with identical field values but different IDs are different objects. An entity's identity persists across time and state changes.

```rust
struct User {
    id: UserId,          // Identity -- this is what makes it unique
    name: String,
    email: String,
}

// These are DIFFERENT users, even with the same name and email
let alice1 = User { id: UserId(1), name: "Alice".into(), email: "alice@test.com".into() };
let alice2 = User { id: UserId(2), name: "Alice".into(), email: "alice@test.com".into() };
assert_ne!(alice1.id, alice2.id);  // Different entities
```

### Value Objects

Value objects have no identity. They are defined entirely by their attributes. Two value objects with the same attributes are interchangeable.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Money {
    amount_cents: i64,
    currency: Currency,
}

impl Money {
    pub fn new(amount_cents: i64, currency: Currency) -> Result<Self, DomainError> {
        // Value objects validate on creation
        if amount_cents < 0 {
            return Err(DomainError::NegativeAmount);
        }
        Ok(Self { amount_cents, currency })
    }

    // Value objects are immutable -- operations return new instances
    pub fn add(&self, other: &Money) -> Result<Money, DomainError> {
        if self.currency != other.currency {
            return Err(DomainError::CurrencyMismatch);
        }
        Money::new(self.amount_cents + other.amount_cents, self.currency)
    }
}
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

```rust
#[derive(Debug, Clone)]
pub enum OrderEvent {
    OrderPlaced {
        order_id: OrderId,
        customer_id: CustomerId,
        total: Money,
        placed_at: DateTime<Utc>,
    },
    OrderShipped {
        order_id: OrderId,
        tracking_number: String,
        shipped_at: DateTime<Utc>,
    },
    OrderCancelled {
        order_id: OrderId,
        reason: CancellationReason,
        cancelled_at: DateTime<Utc>,
    },
}
```

### Why Domain Events Matter

Domain events decouple parts of the system. When an order is placed:
- The **Inventory** context reacts by reserving stock
- The **Email** context sends a confirmation
- The **Analytics** context records the sale
- The **Loyalty** context awards points

The Order context knows none of this. It publishes an event and moves on. New reactions can be added without modifying the Order context.

### Implementation Pattern in Rust

```rust
// Aggregate collects events during operations
impl Order {
    pub fn submit(&mut self) -> Result<Vec<OrderEvent>, OrderError> {
        // ... validation ...
        self.status = OrderStatus::Submitted;
        Ok(vec![OrderEvent::OrderPlaced {
            order_id: self.id.clone(),
            customer_id: self.customer_id.clone(),
            total: self.total.clone(),
            placed_at: Utc::now(),
        }])
    }
}

// Application service publishes events after persisting
impl<R: OrderRepository, E: EventPublisher> OrderService<R, E> {
    pub async fn submit_order(&self, order_id: OrderId) -> Result<(), AppError> {
        let mut order = self.repo.find_by_id(&order_id).await?
            .ok_or(AppError::NotFound)?;

        let events = order.submit()?;
        self.repo.save(&order).await?;

        // Publish after successful persistence
        for event in events {
            self.event_publisher.publish(event).await?;
        }
        Ok(())
    }
}
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
