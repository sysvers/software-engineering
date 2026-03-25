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

```text
/// A projection that builds a flat order list for the admin dashboard.
STRUCTURE OrderListProjection:
    pool ← PgPool

PROCEDURE HANDLE_EVENT(projection, event):
    MATCH event:
        OrderPlaced { order_id, customer_id, items, placed_at } →
            total ← SUM OF item.price FOR EACH item IN items
            AWAIT EXECUTE projection.pool:
                "INSERT INTO order_list_view (id, customer_id, item_count, total, status, placed_at)
                 VALUES (order_id, customer_id, LENGTH(items), total, 'placed', placed_at)"

        OrderShipped { order_id, shipped_at, tracking_number } →
            AWAIT EXECUTE projection.pool:
                "UPDATE order_list_view SET status = 'shipped', tracking = tracking_number,
                 shipped_at = shipped_at WHERE id = order_id"

        OrderDelivered { order_id, delivered_at } →
            AWAIT EXECUTE projection.pool:
                "UPDATE order_list_view SET status = 'delivered', delivered_at = delivered_at
                 WHERE id = order_id"

        _ → // Ignore events this projection does not care about
    RETURN Ok
```

## Multiple Projections from the Same Events

This is where projections become powerful. The same `OrderPlaced`, `OrderShipped`, and `OrderDelivered` events can feed completely different read models.

### Revenue Dashboard Projection

```text
STRUCTURE RevenueDashboardProjection:
    pool ← PgPool

PROCEDURE HANDLE_EVENT(projection, event):
    MATCH event:
        OrderPlaced { placed_at, items, ... } →
            total ← SUM OF item.price FOR EACH item IN items
            month ← FORMAT(placed_at, "YYYY-MM")
            AWAIT EXECUTE projection.pool:
                "INSERT INTO revenue_by_month (month, total_revenue, order_count)
                 VALUES (month, total, 1)
                 ON CONFLICT (month) DO UPDATE
                 SET total_revenue = total_revenue + total,
                     order_count = order_count + 1"
        _ → // ignore
    RETURN Ok
```

### Customer Activity Projection

```text
STRUCTURE CustomerActivityProjection:
    pool ← PgPool

PROCEDURE HANDLE_EVENT(projection, event):
    MATCH event:
        OrderPlaced { customer_id, items, placed_at, ... } →
            total ← SUM OF item.price FOR EACH item IN items
            AWAIT EXECUTE projection.pool:
                "INSERT INTO customer_activity (customer_id, total_spent, order_count, last_order_at)
                 VALUES (customer_id, total, 1, placed_at)
                 ON CONFLICT (customer_id) DO UPDATE
                 SET total_spent = total_spent + total,
                     order_count = order_count + 1,
                     last_order_at = placed_at"
        _ → // ignore
    RETURN Ok
```

Three projections, same events, completely different structures. Adding the fourth projection (say, a geographic sales heatmap) requires no changes to the event producers.

## Rebuilding Projections

Projections are disposable. If a projection has a bug, or requirements change, you can:

1. Drop the projection's tables.
2. Create new tables with the corrected schema.
3. Replay all events from the event store through the new projection logic.

```text
PROCEDURE REBUILD_PROJECTION(store, projection):
    AWAIT projection.RESET()  // DROP and recreate tables

    position ← 0
    LOOP:
        batch ← AWAIT store.READ_ALL_EVENTS(position, 1000)
        IF batch IS empty THEN BREAK
        FOR EACH event IN batch:
            AWAIT projection.HANDLE_EVENT(event)
        position ← position + LENGTH(batch)

INTERFACE Projection:
    PROCEDURE HANDLE_EVENT(event) → Result
    PROCEDURE RESET() → Result
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

```text
PROCEDURE RUN_PROJECTION(store, projection, name, pool):
    last_pos ← AWAIT GET_CHECKPOINT(pool, name) OR DEFAULT 0
    events ← AWAIT store.READ_ALL_EVENTS(last_pos, 500)

    FOR EACH event IN events:
        AWAIT projection.HANDLE_EVENT(event)
        AWAIT SAVE_CHECKPOINT(pool, name, event.position)
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

```text
PROCEDURE LOAD_WITH_SNAPSHOT(store, aggregate_id):
    // Try loading the latest snapshot
    snapshot ← AWAIT QUERY store.pool:
        "SELECT version, state FROM snapshots
         WHERE aggregate_id = aggregate_id
         ORDER BY version DESC LIMIT 1"

    IF snapshot IS present THEN
        aggregate ← PARSE_JSON(snapshot.state) AS Aggregate
        from_version ← snapshot.version
    ELSE
        aggregate ← None
        from_version ← 0

    // Replay only events after the snapshot
    events ← AWAIT LOAD_AFTER_VERSION(store, aggregate_id, from_version)
    FOR EACH event IN events:
        aggregate ← APPLY(aggregate, event)

    RETURN aggregate

PROCEDURE SAVE_SNAPSHOT(store, aggregate):
    state ← TO_JSON(aggregate)
    AWAIT EXECUTE store.pool:
        "INSERT INTO snapshots (aggregate_id, aggregate_type, version, state)
         VALUES (aggregate.id, aggregate.type, aggregate.version, state)
         ON CONFLICT (aggregate_id, version) DO NOTHING"
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
