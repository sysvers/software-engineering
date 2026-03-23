# Projections and Snapshots

## Projections: Building Read Models from Events

If events are the source of truth, projections (also called "read models" or "views") are materialized views built from those events for efficient querying. The event store is optimized for writing; projections are optimized for reading.

```
Events (append-only):
  OrderPlaced -> OrderShipped -> OrderDelivered

Projections (derived from events):
  +------------------------------------------+
  | Order Status View                         |
  | { order_id: 123, status: "delivered" }   |
  +------------------------------------------+
  +------------------------------------------+
  | Revenue Dashboard                         |
  | { date: "2024-03", revenue: 125000 }     |
  +------------------------------------------+
  +------------------------------------------+
  | Customer Order History                    |
  | { customer: "alice", orders: [...] }     |
  +------------------------------------------+
```

**Key insight:** You can build *different* projections from the *same* events, each optimized for a specific query pattern. Need a new dashboard? Build a new projection that replays all historical events -- no schema migration needed.

## How Projections Work

A projection subscribes to events and maintains a read-optimized data structure (typically a database table). Each projection handles only the events it cares about.

```rust
/// A projection that builds a flat order list for the admin dashboard.
struct OrderListProjection {
    pool: PgPool,
}

impl OrderListProjection {
    async fn handle_event(&self, event: &OrderEvent) -> Result<(), ProjectionError> {
        match event {
            OrderEvent::OrderPlaced { order_id, customer_id, items, placed_at } => {
                let total: f64 = items.iter().map(|i| i.price).sum();
                sqlx::query(
                    "INSERT INTO order_list_view (id, customer_id, item_count, total, status, placed_at)
                     VALUES ($1, $2, $3, $4, 'placed', $5)"
                )
                .bind(order_id)
                .bind(customer_id)
                .bind(items.len() as i32)
                .bind(total)
                .bind(placed_at)
                .execute(&self.pool)
                .await?;
            }
            OrderEvent::OrderShipped { order_id, shipped_at, tracking_number } => {
                sqlx::query(
                    "UPDATE order_list_view SET status = 'shipped', tracking = $2, shipped_at = $3
                     WHERE id = $1"
                )
                .bind(order_id)
                .bind(tracking_number)
                .bind(shipped_at)
                .execute(&self.pool)
                .await?;
            }
            OrderEvent::OrderDelivered { order_id, delivered_at } => {
                sqlx::query(
                    "UPDATE order_list_view SET status = 'delivered', delivered_at = $2
                     WHERE id = $1"
                )
                .bind(order_id)
                .bind(delivered_at)
                .execute(&self.pool)
                .await?;
            }
            _ => {} // Ignore events this projection does not care about
        }
        Ok(())
    }
}
```

## Multiple Projections from the Same Events

This is where projections become powerful. The same `OrderPlaced`, `OrderShipped`, and `OrderDelivered` events can feed completely different read models.

### Revenue Dashboard Projection

```rust
struct RevenueDashboardProjection {
    pool: PgPool,
}

impl RevenueDashboardProjection {
    async fn handle_event(&self, event: &OrderEvent) -> Result<(), ProjectionError> {
        match event {
            OrderEvent::OrderPlaced { placed_at, items, .. } => {
                let total: f64 = items.iter().map(|i| i.price).sum();
                let month = placed_at.format("%Y-%m").to_string();
                sqlx::query(
                    "INSERT INTO revenue_by_month (month, total_revenue, order_count)
                     VALUES ($1, $2, 1)
                     ON CONFLICT (month) DO UPDATE
                     SET total_revenue = revenue_by_month.total_revenue + $2,
                         order_count = revenue_by_month.order_count + 1"
                )
                .bind(&month)
                .bind(total)
                .execute(&self.pool)
                .await?;
            }
            _ => {}
        }
        Ok(())
    }
}
```

### Customer Activity Projection

```rust
struct CustomerActivityProjection {
    pool: PgPool,
}

impl CustomerActivityProjection {
    async fn handle_event(&self, event: &OrderEvent) -> Result<(), ProjectionError> {
        match event {
            OrderEvent::OrderPlaced { customer_id, items, placed_at, .. } => {
                let total: f64 = items.iter().map(|i| i.price).sum();
                sqlx::query(
                    "INSERT INTO customer_activity (customer_id, total_spent, order_count, last_order_at)
                     VALUES ($1, $2, 1, $3)
                     ON CONFLICT (customer_id) DO UPDATE
                     SET total_spent = customer_activity.total_spent + $2,
                         order_count = customer_activity.order_count + 1,
                         last_order_at = $3"
                )
                .bind(customer_id)
                .bind(total)
                .bind(placed_at)
                .execute(&self.pool)
                .await?;
            }
            _ => {}
        }
        Ok(())
    }
}
```

Three projections, same events, completely different structures. Adding the fourth projection (say, a geographic sales heatmap) requires no changes to the event producers.

## Rebuilding Projections

Projections are disposable. If a projection has a bug, or requirements change, you can:

1. Drop the projection's tables.
2. Create new tables with the corrected schema.
3. Replay all events from the event store through the new projection logic.

```rust
async fn rebuild_projection(
    store: &EventStore,
    projection: &impl Projection,
) -> Result<(), RebuildError> {
    projection.reset().await?;  // DROP and recreate tables

    let mut position: u64 = 0;
    loop {
        let batch = store.read_all_events(position, 1000).await?;
        if batch.is_empty() {
            break;
        }
        for event in &batch {
            projection.handle_event(event).await?;
        }
        position += batch.len() as u64;
    }

    Ok(())
}

trait Projection {
    async fn handle_event(&self, event: &StoredEvent) -> Result<(), ProjectionError>;
    async fn reset(&self) -> Result<(), ProjectionError>;
}
```

**Important consideration:** Rebuilding a projection from millions of events takes time. Plan for this. Options include:

- Run rebuilds during off-peak hours.
- Build new projections alongside old ones, then switch reads once the rebuild is complete.
- Partition events by time to enable parallel replay.

## Tracking Projection Position

Each projection must track which event it has processed last, so it can resume after a restart without reprocessing everything.

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_position   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```rust
async fn run_projection(
    store: &EventStore,
    projection: &impl Projection,
    name: &str,
    pool: &PgPool,
) -> Result<(), ProjectionError> {
    let last_pos = get_checkpoint(pool, name).await?.unwrap_or(0);
    let events = store.read_all_events(last_pos, 500).await?;

    for event in &events {
        projection.handle_event(event).await?;
        save_checkpoint(pool, name, event.position).await?;
    }
    Ok(())
}
```

## Snapshots

Replaying thousands of events to rebuild an aggregate is slow. Snapshots periodically save the current state, so you only replay events *after* the snapshot.

```
Without snapshot: Replay events 1-10,000 -> Current state
With snapshot:    Load snapshot @event 9,950 -> Replay events 9,951-10,000 -> Current state
```

### Snapshot Storage

```sql
CREATE TABLE snapshots (
    aggregate_id    UUID NOT NULL,
    aggregate_type  TEXT NOT NULL,
    version         INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (aggregate_id, version)
);
```

### Rust Snapshot Implementation

```rust
impl EventStore {
    async fn load_with_snapshot<A: Aggregate + DeserializeOwned>(
        &self,
        aggregate_id: Uuid,
    ) -> Result<Option<A>, EventStoreError> {
        // Try loading the latest snapshot
        let snapshot = sqlx::query!(
            "SELECT version, state FROM snapshots
             WHERE aggregate_id = $1
             ORDER BY version DESC LIMIT 1",
            aggregate_id
        )
        .fetch_optional(&self.pool)
        .await?;

        let (mut aggregate, from_version) = match snapshot {
            Some(snap) => {
                let agg: A = serde_json::from_value(snap.state)?;
                (Some(agg), snap.version)
            }
            None => (None, 0),
        };

        // Replay only events after the snapshot
        let events = self.load_after_version(aggregate_id, from_version).await?;
        for event in &events {
            aggregate = A::apply(aggregate, event);
        }

        Ok(aggregate)
    }

    async fn save_snapshot<A: Aggregate + Serialize>(
        &self,
        aggregate: &A,
    ) -> Result<(), EventStoreError> {
        let state = serde_json::to_value(aggregate)?;
        sqlx::query(
            "INSERT INTO snapshots (aggregate_id, aggregate_type, version, state)
             VALUES ($1, $2, $3, $4)
             ON CONFLICT (aggregate_id, version) DO NOTHING"
        )
        .bind(aggregate.id())
        .bind(A::aggregate_type())
        .bind(aggregate.version())
        .bind(&state)
        .execute(&self.pool)
        .await?;
        Ok(())
    }
}
```

### When to Snapshot

- **Every N events:** Take a snapshot every 100 or 500 events. Simple and predictable.
- **On performance threshold:** Snapshot when replay time exceeds a threshold.
- **Never (for short-lived aggregates):** If an aggregate only accumulates 10-20 events over its lifetime, snapshots add complexity for no benefit.

### Snapshot Pitfalls

- **Snapshot schema drift:** If you change the aggregate structure, old snapshots may not deserialize. Version your snapshots or fall back to full replay on deserialization failure.
- **Snapshots are not the source of truth.** They are an optimization. If a snapshot is corrupted, delete it and replay from events.
- **Do not snapshot projections.** Projections track their position via checkpoints. If they get corrupted, rebuild from events.

## Key Takeaways

1. Projections are read models derived from events, each optimized for a specific query pattern.
2. Multiple projections from the same events enable different views without changing producers.
3. Projections are disposable -- rebuild them by replaying events when logic or schema changes.
4. Track projection position with checkpoints so they can resume after restarts.
5. Snapshots optimize aggregate loading by avoiding full event replay, but they are not the source of truth.
6. Snapshot every N events for aggregates with long histories. Skip them for short-lived aggregates.
