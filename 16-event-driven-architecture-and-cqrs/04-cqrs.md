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

```text
// Commands -- requests to change state
ENUMERATION OrderCommand:
    PlaceOrder { customer_id, items }
    CancelOrder { order_id, reason }
    ShipOrder { order_id, tracking_number }

// Events -- results of successful commands
ENUMERATION OrderEvent:
    OrderPlaced { order_id, customer_id, items, placed_at }
    OrderCancelled { order_id, reason, cancelled_at }
    OrderShipped { order_id, tracking_number, shipped_at }

// Write model -- enforces business rules
STRUCTURE OrderAggregate:
    id ← OrderId
    status ← OrderStatus
    items ← list of OrderItem
    customer_id ← string
    version ← integer

PROCEDURE HANDLE_COMMAND(aggregate, cmd):
    MATCH cmd:
        PlaceOrder { items, customer_id } →
            IF items IS empty THEN
                RETURN Error(EmptyOrder)
            IF LENGTH(items) > 100 THEN
                RETURN Error(TooManyItems)
            RETURN [OrderPlaced {
                order_id ← NEW_ORDER_ID(), customer_id,
                items, placed_at ← NOW()
            }]

        CancelOrder { order_id, reason } →
            IF aggregate.status = Shipped THEN
                RETURN Error(CannotCancelShippedOrder)
            IF aggregate.status = Cancelled THEN
                RETURN Error(AlreadyCancelled)
            RETURN [OrderCancelled {
                order_id, reason, cancelled_at ← NOW()
            }]

        ShipOrder { order_id, tracking_number } →
            IF aggregate.status ≠ Placed THEN
                RETURN Error(InvalidStatusForShipping)
            RETURN [OrderShipped {
                order_id, tracking_number, shipped_at ← NOW()
            }]
```

### Read Side: Projections for Queries

```text
// Read model -- optimized for queries, completely denormalized
STRUCTURE OrderListView:
    pool ← PgPool

/// Update the read model when events occur.
PROCEDURE ON_EVENT(view, event):
    MATCH event:
        OrderPlaced { order_id, customer_id, items, placed_at } →
            AWAIT EXECUTE view.pool:
                "INSERT INTO order_list_view (id, customer_id, item_count, total, status, placed_at)
                 VALUES (order_id, customer_id, LENGTH(items), SUM(item.price), 'placed', placed_at)"

        OrderCancelled { order_id, cancelled_at, ... } →
            AWAIT EXECUTE view.pool:
                "UPDATE order_list_view SET status = 'cancelled', cancelled_at = cancelled_at
                 WHERE id = order_id"

        OrderShipped { order_id, tracking_number, shipped_at } →
            AWAIT EXECUTE view.pool:
                "UPDATE order_list_view SET status = 'shipped', tracking = tracking_number,
                 shipped_at = shipped_at WHERE id = order_id"
    RETURN Ok

/// Fast queries against the denormalized read model.
PROCEDURE GET_CUSTOMER_ORDERS(view, customer_id):
    RETURN AWAIT QUERY view.pool:
        "SELECT id, item_count, total, status, placed_at
         FROM order_list_view WHERE customer_id = customer_id
         ORDER BY placed_at DESC LIMIT 50"

PROCEDURE SEARCH_ORDERS(view, query):
    RETURN AWAIT QUERY view.pool:
        "SELECT id, item_count, total, status, placed_at
         FROM order_list_view WHERE status = query
         ORDER BY placed_at DESC"
```

## Eventual Consistency

In CQRS, the read model is *eventually consistent* with the write model. After a command produces events, there is a delay before projections process those events and update the read model.

### Handling Eventual Consistency in Practice

**Return written data directly.** After a write, return the result to the client without waiting for the projection to update. The user sees their action confirmed immediately.

```text
PROCEDURE PLACE_ORDER(cmd):
    events ← HANDLE_COMMAND(aggregate, cmd)
    AWAIT store.APPEND(events)

    // Return the result directly -- don't query the read model
    RETURN HTTP 200 JSON {
        order_id ← events[0].order_id,
        status ← "placed"
    }
```

**Optimistic UI.** The frontend updates immediately and reconciles later. When a user clicks "Cancel Order," the UI shows it as cancelled before the backend confirms.

**Causal consistency.** Include a version or timestamp so clients know if their read is "fresh enough."

```text
PROCEDURE GET_ORDERS(request):
    min_version ← request.HEADER("X-Min-Version") OR DEFAULT 0
    view ← AWAIT order_view.GET_CUSTOMER_ORDERS(request.customer_id)

    // If the read model hasn't caught up, tell the client to retry
    IF view.version < min_version THEN
        RETURN HTTP 503 with header "Retry-After" ← "1"

    RETURN HTTP 200 JSON(view)
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
