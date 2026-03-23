# NoSQL Data Modeling

NoSQL databases demand a fundamentally different mindset from relational databases. Instead of normalizing data and joining at query time, you denormalize aggressively and model your tables around access patterns. This document covers the three most commonly used NoSQL categories in production systems, with concrete modeling patterns and real migration stories.

---

## DynamoDB Single-Table Design

DynamoDB charges per read/write and has no JOIN operator. The dominant pattern at scale is **single-table design**: store all entity types in one table with carefully designed partition keys (PK) and sort keys (SK).

### The core idea

Instead of separate tables for users, orders, and items, you encode entity type and relationships into the key structure:

```
PK                | SK                  | GSI1-PK          | GSI1-SK            | Attributes
USER#u_123        | PROFILE             |                  |                    | { name, email, plan }
USER#u_123        | ORDER#2024-03-15#o1 | ORDER#o1         | USER#u_123         | { total: 89.99, status: "shipped" }
USER#u_123        | ORDER#2024-03-20#o2 | ORDER#o2         | USER#u_123         | { total: 149.50, status: "pending" }
ORDER#o1          | ITEM#i_001          | PRODUCT#p_widget | ORDER#o1           | { product: "Widget", qty: 2, price: 29.99 }
ORDER#o1          | ITEM#i_002          | PRODUCT#p_gadget | ORDER#o1           | { product: "Gadget", qty: 1, price: 30.01 }
```

### Access patterns this supports

| Access pattern | Query |
|----------------|-------|
| Get user profile | `PK = USER#u_123, SK = PROFILE` |
| List user's orders (newest first) | `PK = USER#u_123, SK begins_with ORDER#` (reverse sort) |
| Get all items in an order | `PK = ORDER#o1, SK begins_with ITEM#` |
| Find order by order ID | GSI1: `PK = ORDER#o1` |
| Find all orders for a product | GSI1: `PK = PRODUCT#p_widget` |

### Designing for DynamoDB

1. **List every access pattern first.** Write them all down before you touch the schema. DynamoDB tables cannot be efficiently queried in ways you did not plan for.
2. **Choose a partition key with high cardinality.** User IDs, order IDs, device IDs. Never use something with low cardinality like `status` or `country` — that creates hot partitions.
3. **Use sort keys for hierarchical or time-based queries.** Embedding timestamps in sort keys gives you free chronological ordering.
4. **Add Global Secondary Indexes (GSIs) for alternate access patterns.** Each GSI is essentially a full copy of the table with a different key layout. Keep the count low (typically 1-3).
5. **Use composite sort keys for filtering.** `STATUS#shipped#2024-03-15` lets you query by status prefix and then by date range within that status.

### When single-table design breaks down

Single-table design is powerful but has limits:

- **Ad-hoc queries are nearly impossible.** If product management asks "which users in France ordered more than $500 last quarter," you cannot answer that without a full table scan or a separate analytics system.
- **Capacity planning is harder.** All entities share the same provisioned throughput.
- **Code complexity increases.** Your application must encode/decode the key structure and handle polymorphic items from the same table.

For analytics and reporting, export DynamoDB data to S3 and query with Athena, or replicate to a relational warehouse.

---

## Redis Patterns

Redis is an in-memory data structure server. It is not a primary database — it is a cache, a coordination layer, and a real-time data structure engine.

### Caching patterns

**Cache-aside (lazy loading):** The application checks the cache first. On a miss, it queries the database, then populates the cache.

```rust
use redis::AsyncCommands;

async fn get_product(
    redis: &mut redis::aio::MultiplexedConnection,
    db: &sqlx::PgPool,
    product_id: i64,
) -> anyhow::Result<Product> {
    let cache_key = format!("product:{}", product_id);

    // Try cache
    if let Some(json) = redis.get::<_, Option<String>>(&cache_key).await? {
        return Ok(serde_json::from_str(&json)?);
    }

    // Cache miss — fetch from PostgreSQL
    let product = sqlx::query_as::<_, Product>(
        "SELECT id, name, price_cents, stock FROM products WHERE id = $1"
    )
    .bind(product_id)
    .fetch_one(db)
    .await?;

    // Populate cache with 10-minute TTL
    let json = serde_json::to_string(&product)?;
    redis.set_ex(&cache_key, &json, 600).await?;

    Ok(product)
}
```

**Write-through:** Every write goes to both the database and the cache. Ensures cache freshness at the cost of write latency.

**Cache invalidation:** When data changes, delete the cache key rather than trying to update it. The next read repopulates with fresh data. This avoids race conditions between concurrent writers.

```rust
async fn update_product_price(
    redis: &mut redis::aio::MultiplexedConnection,
    db: &sqlx::PgPool,
    product_id: i64,
    new_price: i64,
) -> anyhow::Result<()> {
    sqlx::query("UPDATE products SET price_cents = $1 WHERE id = $2")
        .bind(new_price)
        .bind(product_id)
        .execute(db)
        .await?;

    // Invalidate — do not update
    redis.del(format!("product:{}", product_id)).await?;
    Ok(())
}
```

### Sorted sets for leaderboards and ranking

Redis sorted sets maintain elements ordered by score with O(log N) insertion and O(log N) rank lookups.

```
ZADD leaderboard 1500 "player:alice"
ZADD leaderboard 2300 "player:bob"
ZADD leaderboard 1800 "player:carol"

ZREVRANGE leaderboard 0 9 WITHSCORES   -- Top 10 players
ZRANK leaderboard "player:alice"         -- Alice's rank (0-indexed)
ZINCRBY leaderboard 100 "player:alice"   -- Alice scored 100 more points
```

Real-world example: a gaming platform with 10 million players. PostgreSQL cannot return "top 100 players" in under 1ms because it requires sorting the entire table (even with an index, reading 10M rows is slow). A Redis sorted set answers this in microseconds.

### Pub/sub for real-time events

Redis pub/sub delivers messages to all subscribers on a channel. Messages are fire-and-forget — no persistence, no acknowledgment.

```rust
// Publisher
redis.publish("notifications:user:123", "You have a new order").await?;

// Subscriber (runs in a separate task)
let mut pubsub = redis_conn.into_pubsub();
pubsub.subscribe("notifications:user:123").await?;

while let Some(msg) = pubsub.on_message().next().await {
    let payload: String = msg.get_payload()?;
    println!("Received: {}", payload);
}
```

**When to use pub/sub vs. Redis Streams:** Pub/sub is ephemeral — if no subscriber is listening, the message is lost. Redis Streams (`XADD`, `XREAD`, `XREADGROUP`) provide persistent, consumer-group-based messaging similar to Kafka. Use Streams when you need durability and at-least-once delivery.

### Rate limiting

```
-- Allow 100 requests per minute per IP
MULTI
INCR   ratelimit:192.168.1.1:202403241530
EXPIRE ratelimit:192.168.1.1:202403241530 60
EXEC
-- If the counter exceeds 100, reject the request
```

---

## MongoDB Document Modeling

MongoDB stores data as BSON documents (binary JSON). The key modeling decision is **embedding vs. referencing**.

### Embedding (denormalized)

Store related data inside the parent document.

```json
{
    "_id": "order_456",
    "user_id": "user_123",
    "status": "shipped",
    "items": [
        { "product": "Widget", "qty": 2, "price": 29.99 },
        { "product": "Gadget", "qty": 1, "price": 30.01 }
    ],
    "shipping_address": {
        "street": "123 Main St",
        "city": "Portland",
        "state": "OR",
        "zip": "97201"
    },
    "created_at": "2024-03-15T10:30:00Z"
}
```

**Embed when:**
- The related data is always accessed together with the parent
- The related data does not change independently
- The embedded array has a bounded, small size (16 MB document limit)

### Referencing (normalized)

Store a reference (ID) and look it up separately.

```json
// users collection
{ "_id": "user_123", "name": "Alice", "email": "alice@example.com" }

// orders collection
{ "_id": "order_456", "user_id": "user_123", "total": 89.99 }
```

**Reference when:**
- The related entity is shared across many parents (a product referenced by thousands of orders)
- The related data is large or unbounded
- The related data is updated independently

### The bucket pattern (time-series)

Instead of one document per event, group events into time-based buckets.

```json
{
    "_id": "sensor_42:2024-03-15:14",
    "sensor_id": "sensor_42",
    "hour": "2024-03-15T14:00:00Z",
    "readings": [
        { "t": "14:00:05", "temp": 22.3, "humidity": 45 },
        { "t": "14:00:10", "temp": 22.4, "humidity": 44 },
        // ... up to ~200 readings per bucket
    ],
    "count": 200,
    "avg_temp": 22.35
}
```

This reduces the number of documents by orders of magnitude and improves query performance for time-range queries.

---

## When NoSQL Beats SQL

| Scenario | Why NoSQL wins |
|----------|---------------|
| Massive write throughput (>100K writes/sec) | Cassandra/ScyllaDB distribute writes across a cluster with no single-leader bottleneck |
| Simple key-value access at low latency | DynamoDB and Redis deliver single-digit millisecond reads predictably |
| Schema varies per record | MongoDB lets each document have different fields without ALTER TABLE |
| Horizontal scaling with no downtime | DynamoDB auto-scales; Cassandra adds nodes with zero downtime |
| Caching and ephemeral data | Redis provides sub-millisecond access for data that does not need durability |

## When SQL Beats NoSQL

| Scenario | Why SQL wins |
|----------|-------------|
| Complex queries with joins and aggregations | PostgreSQL handles multi-table joins, window functions, CTEs natively |
| Strong consistency across entities | ACID transactions guarantee atomicity; DynamoDB transactions are limited |
| Ad-hoc reporting | SQL is the universal query language for analytics; NoSQL requires export pipelines |
| Small-to-medium data volumes | PostgreSQL on a single node handles hundreds of millions of rows with proper indexing |
| Evolving query patterns | Adding a new WHERE clause in SQL is trivial; in DynamoDB it may require a new GSI or table redesign |

---

## Case Study: Discord's Database Migration Journey

Discord's migration path is one of the best-documented examples of choosing databases based on evolving access patterns.

### Phase 1: MongoDB (2015)

Discord started with MongoDB for rapid prototyping. Document storage made sense for messages — each message is a self-contained unit. MongoDB worked well at small scale.

**What broke:** As Discord grew to millions of users, MongoDB struggled with their access pattern: "fetch the most recent messages in a channel." Random I/O patterns on MongoDB caused unpredictable latency spikes. Their data set exceeded available RAM, and MongoDB's memory-mapped storage engine performed poorly when reads hit disk.

### Phase 2: Cassandra (2017)

Discord migrated to Cassandra. The data model was natural: partition by `(channel_id, bucket)` where bucket is a time window, and sort by `message_id` within each partition. This gave efficient range queries for "recent messages in channel X."

**What broke:** Cassandra's JVM-based architecture caused garbage collection pauses that spiked tail latency. At Discord's scale (billions of messages, thousands of concurrent reads), GC pauses at the 99th percentile were unacceptable for a real-time chat application.

### Phase 3: ScyllaDB (2022)

Discord migrated to ScyllaDB, a Cassandra-compatible database written in C++ with a shard-per-core architecture. Same data model, same query language (CQL), but no GC pauses. P99 latency dropped dramatically.

**Key takeaways:**
- Each migration was driven by a specific, measured performance problem — not hype
- The data model (partition by channel, sort by time) stayed consistent across Cassandra and ScyllaDB because the access pattern did not change
- Discord runs PostgreSQL alongside ScyllaDB for relational data (user accounts, guilds, permissions). NoSQL did not replace SQL — it complemented it

---

## Pitfalls

- **Designing DynamoDB tables like relational tables.** Multiple tables with "joins" in application code negates DynamoDB's strengths. Invest time in single-table design up front.
- **Using MongoDB for highly relational data.** If you need "show all users who purchased products in category X," MongoDB will fight you. Use PostgreSQL.
- **Treating Redis as durable.** Even with AOF persistence, Redis can lose the last second of writes on crash. Never store data in Redis that you cannot regenerate.
- **Ignoring hot partitions in DynamoDB.** A partition key like `status` with three possible values creates three hot partitions. Use high-cardinality keys.
- **Unbounded arrays in MongoDB documents.** A chat room document with an embedded `messages` array will hit the 16 MB limit and degrade performance long before that. Reference or bucket instead.
- **Not planning for analytics.** NoSQL databases are terrible at ad-hoc queries. Build an export pipeline to a data warehouse from day one if you need reporting.
