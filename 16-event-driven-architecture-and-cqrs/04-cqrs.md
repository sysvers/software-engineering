# CQRS (Command Query Responsibility Segregation)

## The Core Idea

CQRS separates the write model (commands) from the read model (queries). Instead of one model that handles both reads and writes, you have two specialized models.

```
                    +------------------+
   Commands ------> |   Write Model    | --> Event Store
   (Create, Update) |   (Domain Logic) |         |
                    +------------------+         |
                                                  | Events
                                                  v
                    +------------------+    +----------+
   Queries -------> |   Read Model     | <--|Projection|
   (List, Search)   | (Optimized Views)|    | Builder  |
                    +------------------+    +----------+
```

## Why Separate Reads from Writes?

### 1. Different Optimization Needs

Writes need strong consistency, validation, and business rules. Reads need fast queries, denormalized data, and caching. A single model forces compromises on both sides.

With a traditional ORM, you normalize data for write integrity, then write complex JOINs to reassemble it for reads. With CQRS, the write side stays normalized and the read side is pre-joined and denormalized.

### 2. Independent Scaling

Most systems are read-heavy (often 100:1 read-to-write ratio). CQRS lets you scale read replicas without affecting write performance. The read database can be a completely different technology -- PostgreSQL for writes, Elasticsearch for full-text search reads, Redis for hot dashboard data.

### 3. Simpler Models

The write model focuses on invariants and business rules. The read model focuses on query performance. Neither is polluted by the other's concerns. The write model does not need `ORDER BY` or pagination. The read model does not need validation logic.

## Rust Implementation

### Write Side: Command Handling

```rust
// Commands -- requests to change state
enum OrderCommand {
    PlaceOrder { customer_id: String, items: Vec<OrderItem> },
    CancelOrder { order_id: OrderId, reason: String },
    ShipOrder { order_id: OrderId, tracking_number: String },
}

// Events -- results of successful commands
enum OrderEvent {
    OrderPlaced { order_id: OrderId, customer_id: String, items: Vec<OrderItem>, placed_at: DateTime<Utc> },
    OrderCancelled { order_id: OrderId, reason: String, cancelled_at: DateTime<Utc> },
    OrderShipped { order_id: OrderId, tracking_number: String, shipped_at: DateTime<Utc> },
}

// Write model -- enforces business rules
struct OrderAggregate {
    id: OrderId,
    status: OrderStatus,
    items: Vec<OrderItem>,
    customer_id: String,
    version: i32,
}

impl OrderAggregate {
    fn handle_command(&self, cmd: OrderCommand) -> Result<Vec<OrderEvent>, OrderError> {
        match cmd {
            OrderCommand::PlaceOrder { items, customer_id } => {
                if items.is_empty() {
                    return Err(OrderError::EmptyOrder);
                }
                if items.len() > 100 {
                    return Err(OrderError::TooManyItems);
                }
                Ok(vec![OrderEvent::OrderPlaced {
                    order_id: OrderId::new(),
                    customer_id,
                    items,
                    placed_at: Utc::now(),
                }])
            }
            OrderCommand::CancelOrder { order_id, reason } => {
                if self.status == OrderStatus::Shipped {
                    return Err(OrderError::CannotCancelShippedOrder);
                }
                if self.status == OrderStatus::Cancelled {
                    return Err(OrderError::AlreadyCancelled);
                }
                Ok(vec![OrderEvent::OrderCancelled {
                    order_id,
                    reason,
                    cancelled_at: Utc::now(),
                }])
            }
            OrderCommand::ShipOrder { order_id, tracking_number } => {
                if self.status != OrderStatus::Placed {
                    return Err(OrderError::InvalidStatusForShipping);
                }
                Ok(vec![OrderEvent::OrderShipped {
                    order_id,
                    tracking_number,
                    shipped_at: Utc::now(),
                }])
            }
        }
    }
}
```

### Read Side: Projections for Queries

```rust
// Read model -- optimized for queries, completely denormalized
struct OrderListView {
    pool: PgPool,
}

impl OrderListView {
    /// Update the read model when events occur.
    async fn on_event(&self, event: &OrderEvent) -> Result<(), Error> {
        match event {
            OrderEvent::OrderPlaced { order_id, customer_id, items, placed_at } => {
                sqlx::query!(
                    "INSERT INTO order_list_view (id, customer_id, item_count, total, status, placed_at)
                     VALUES ($1, $2, $3, $4, 'placed', $5)",
                    order_id.as_ref(), customer_id, items.len() as i32,
                    items.iter().map(|i| i.price).sum::<f64>(),
                    placed_at
                ).execute(&self.pool).await?;
            }
            OrderEvent::OrderCancelled { order_id, cancelled_at, .. } => {
                sqlx::query!(
                    "UPDATE order_list_view SET status = 'cancelled', cancelled_at = $2
                     WHERE id = $1",
                    order_id.as_ref(), cancelled_at
                ).execute(&self.pool).await?;
            }
            OrderEvent::OrderShipped { order_id, tracking_number, shipped_at } => {
                sqlx::query!(
                    "UPDATE order_list_view SET status = 'shipped', tracking = $2, shipped_at = $3
                     WHERE id = $1",
                    order_id.as_ref(), tracking_number, shipped_at
                ).execute(&self.pool).await?;
            }
        }
        Ok(())
    }

    /// Fast queries against the denormalized read model.
    async fn get_customer_orders(&self, customer_id: &str) -> Result<Vec<OrderSummary>, Error> {
        sqlx::query_as!(OrderSummary,
            "SELECT id, item_count, total, status, placed_at
             FROM order_list_view WHERE customer_id = $1
             ORDER BY placed_at DESC LIMIT 50",
            customer_id
        ).fetch_all(&self.pool).await
    }

    async fn search_orders(&self, query: &str) -> Result<Vec<OrderSummary>, Error> {
        sqlx::query_as!(OrderSummary,
            "SELECT id, item_count, total, status, placed_at
             FROM order_list_view WHERE status = $1
             ORDER BY placed_at DESC",
            query
        ).fetch_all(&self.pool).await
    }
}
```

## Eventual Consistency

In CQRS, the read model is *eventually consistent* with the write model. After a command produces events, there is a delay before projections process those events and update the read model.

### Handling Eventual Consistency in Practice

**Return written data directly.** After a write, return the result to the client without waiting for the projection to update. The user sees their action confirmed immediately.

```rust
async fn place_order(cmd: PlaceOrderCommand) -> HttpResponse {
    let events = aggregate.handle_command(cmd)?;
    store.append(events.clone()).await?;

    // Return the result directly -- don't query the read model
    HttpResponse::Ok().json(OrderConfirmation {
        order_id: events[0].order_id(),
        status: "placed",
    })
}
```

**Optimistic UI.** The frontend updates immediately and reconciles later. When a user clicks "Cancel Order," the UI shows it as cancelled before the backend confirms.

**Causal consistency.** Include a version or timestamp so clients know if their read is "fresh enough."

```rust
async fn get_orders(req: Request) -> HttpResponse {
    let min_version = req.header("X-Min-Version").unwrap_or(0);
    let view = order_view.get_customer_orders(&req.customer_id).await?;

    // If the read model hasn't caught up, tell the client to retry
    if view.version < min_version {
        return HttpResponse::ServiceUnavailable()
            .header("Retry-After", "1")
            .finish();
    }

    HttpResponse::Ok().json(view)
}
```

**Read-your-writes.** Route the writing user's subsequent reads to the write model or a guaranteed-consistent read replica.

## When CQRS Is Overkill

CQRS adds real complexity: two models to maintain, eventual consistency to handle, projection infrastructure to operate. Do not use it when:

- **Simple CRUD with uniform access patterns.** If your reads and writes use the same data shape, one model is simpler and correct.
- **Low traffic.** If you are not bottlenecked on reads or writes, separate scaling provides no benefit.
- **Small team.** CQRS increases the surface area of the system. A team of two maintaining separate read/write models, projections, and event infrastructure may spend more time on plumbing than features.
- **No complex read patterns.** If all your queries are simple lookups by primary key, a denormalized read model adds nothing.

### The Pragmatic Middle Ground

You do not need full CQRS to get some of its benefits:

- **Read replicas** give you independent read scaling without separate models.
- **Materialized views** in PostgreSQL give you denormalized reads without projection infrastructure.
- **Caching** (Redis, CDN) solves most read performance problems without architectural change.

Reserve full CQRS for systems where read and write models are fundamentally different in structure, scaling needs, or consistency requirements.

## Trade-offs Summary

| Aspect | Single Model | CQRS |
|--------|-------------|------|
| Complexity | Lower | Higher -- two models, eventual consistency |
| Read optimization | Compromised by write concerns | Fully optimized, denormalized |
| Write optimization | Compromised by read concerns | Focused on business rules |
| Scaling | Scale together | Scale independently |
| Consistency | Immediate | Eventually consistent reads |
| Team size | Any | Needs enough people to maintain both sides |

## Key Takeaways

1. CQRS separates read and write models so each can be optimized independently.
2. The write side enforces business rules and produces events. The read side consumes events and builds query-optimized views.
3. Eventual consistency is the trade-off. Plan for it explicitly in your UI and API design.
4. CQRS is not an all-or-nothing choice. Apply it to the parts of your system where read/write patterns diverge significantly.
5. For simple CRUD, CQRS adds cost without proportional benefit. Use simpler alternatives first.
