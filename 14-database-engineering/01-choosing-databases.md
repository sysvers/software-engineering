# Choosing the Right Database

There is no universally "best" database. The right choice depends on your data model, access patterns, consistency requirements, and operational maturity. This document walks through every major category, when each shines, and when it falls apart.

---

## Relational Databases (SQL)

**Examples:** PostgreSQL, MySQL, MariaDB, CockroachDB, SQLite

Relational databases store data in tables with rows and columns, enforce schemas, and support ACID transactions. They remain the default choice for the vast majority of applications.

### Why PostgreSQL is the default

PostgreSQL supports relational data, JSONB for semi-structured data, full-text search, geospatial queries (PostGIS), window functions, CTEs, materialized views, and custom types. It scales vertically to surprisingly large workloads before you need to shard.

**Real-world use case:** A SaaS billing platform with users, subscriptions, invoices, and payments. These entities have strong relationships (a subscription belongs to a user, an invoice has many line items). ACID transactions guarantee that creating an invoice and updating the subscription status happen atomically.

```rust
use sqlx::PgPool;

#[derive(sqlx::FromRow)]
struct Invoice {
    id: i64,
    user_id: i64,
    total_cents: i64,
    status: String,
    created_at: chrono::DateTime<chrono::Utc>,
}

async fn create_invoice_with_line_items(
    pool: &PgPool,
    user_id: i64,
    items: &[(String, i64)],
) -> Result<Invoice, sqlx::Error> {
    let mut tx = pool.begin().await?;

    let total: i64 = items.iter().map(|(_, cents)| cents).sum();

    let invoice = sqlx::query_as::<_, Invoice>(
        "INSERT INTO invoices (user_id, total_cents, status)
         VALUES ($1, $2, 'pending')
         RETURNING id, user_id, total_cents, status, created_at"
    )
    .bind(user_id)
    .bind(total)
    .fetch_one(&mut *tx)
    .await?;

    for (description, amount_cents) in items {
        sqlx::query(
            "INSERT INTO invoice_line_items (invoice_id, description, amount_cents)
             VALUES ($1, $2, $3)"
        )
        .bind(invoice.id)
        .bind(description)
        .bind(amount_cents)
        .execute(&mut *tx)
        .await?;
    }

    tx.commit().await?;
    Ok(invoice)
}
```

### When to pick MySQL over PostgreSQL

MySQL has a larger legacy footprint. Choose it when your team already operates MySQL at scale, when you need MySQL-specific replication topologies, or when a vendor requirement mandates it. For greenfield projects, PostgreSQL is almost always the better choice due to its richer feature set and standards compliance.

### SQLite for embedded and local-first

SQLite is not a server. It is a library that reads and writes directly to a file. Use it for mobile apps, desktop apps, CLI tools, unit tests, and edge computing. Litestream and LiteFS extend it with replication.

---

## Document Databases

**Examples:** MongoDB, CouchDB, Firestore, Amazon DocumentDB

Document databases store data as JSON-like documents (BSON in MongoDB). Each document can have a different structure — no predefined schema required.

### When documents win

- **Content management systems** where each page type has different fields
- **Product catalogs** where a laptop has different attributes than a T-shirt
- **Rapid prototyping** where the schema evolves weekly
- **Event storage** where each event type carries different payloads

### When documents lose

Documents fall apart when you need to query across relationships. "Show me all users who purchased products from category X in the last 30 days" requires multiple collection lookups with no JOIN support. You end up denormalizing aggressively or building the joins in application code.

**Real-world pitfall:** A startup builds their entire e-commerce platform on MongoDB. Six months later, finance needs revenue reports broken down by product category, region, and time period. These cross-collection aggregations are painful. They either build a separate analytics pipeline or migrate to PostgreSQL.

```rust
// Using the mongodb crate to insert a flexible document
use mongodb::{Client, bson::doc};

async fn store_product(client: &Client, product: serde_json::Value) -> mongodb::error::Result<()> {
    let db = client.database("catalog");
    let collection = db.collection::<mongodb::bson::Document>("products");

    // Each product can have different fields
    let doc = mongodb::bson::to_document(&product)?;
    collection.insert_one(doc).await?;
    Ok(())
}
```

---

## Key-Value Stores

**Examples:** Redis, Memcached, Amazon DynamoDB, etcd

Key-value stores map keys to values with O(1) lookups. They are the fastest category for simple read/write patterns.

### Redis: more than a cache

Redis supports strings, hashes, lists, sets, sorted sets, streams, and HyperLogLog. Common patterns:

| Pattern | How |
|---------|-----|
| **Session storage** | `SET session:{token} {user_json} EX 3600` |
| **Rate limiting** | `INCR ratelimit:{ip}:{minute}` with TTL |
| **Leaderboard** | Sorted set: `ZADD leaderboard {score} {user_id}` |
| **Distributed lock** | `SET lock:{resource} {owner} NX EX 30` |
| **Pub/sub** | `PUBLISH channel message` / `SUBSCRIBE channel` |
| **Queue** | `LPUSH queue task` / `BRPOP queue 0` |

```rust
use redis::AsyncCommands;

async fn cache_user_profile(
    conn: &mut redis::aio::MultiplexedConnection,
    user_id: i64,
    profile_json: &str,
) -> redis::RedisResult<()> {
    // Cache for 1 hour
    conn.set_ex(
        format!("user:profile:{}", user_id),
        profile_json,
        3600,
    ).await?;
    Ok(())
}

async fn get_or_fetch_profile(
    redis_conn: &mut redis::aio::MultiplexedConnection,
    pg_pool: &sqlx::PgPool,
    user_id: i64,
) -> anyhow::Result<UserProfile> {
    let cache_key = format!("user:profile:{}", user_id);

    // Try cache first
    let cached: Option<String> = redis_conn.get(&cache_key).await?;
    if let Some(json) = cached {
        return Ok(serde_json::from_str(&json)?);
    }

    // Cache miss: fetch from PostgreSQL
    let profile = sqlx::query_as::<_, UserProfile>(
        "SELECT id, name, email, avatar_url FROM users WHERE id = $1"
    )
    .bind(user_id)
    .fetch_one(pg_pool)
    .await?;

    // Populate cache
    let json = serde_json::to_string(&profile)?;
    redis_conn.set_ex(&cache_key, &json, 3600).await?;

    Ok(profile)
}
```

### Pitfall: Redis as a primary database

Redis stores data in memory. Even with persistence (RDB snapshots, AOF), a crash can lose recent writes. Use Redis for data you can rebuild (caches, rate limits, sessions). Never use it as your source of truth.

---

## Column-Family Databases

**Examples:** Apache Cassandra, ScyllaDB, HBase

Column-family databases are designed for massive write throughput and horizontal scaling. Data is partitioned across nodes using consistent hashing.

### When to choose Cassandra or ScyllaDB

- **Time-series data:** IoT sensor readings, application metrics, event logs
- **Write-heavy workloads:** Millions of writes per second
- **Multi-region deployments:** Built-in replication across data centers
- **Append-only patterns:** Immutable event logs, audit trails

### The trade-offs are real

- **No joins.** You must denormalize everything.
- **Eventual consistency by default.** Tunable, but strong consistency is expensive.
- **Data modeling is hard.** You design tables around queries, not entities. One entity may appear in multiple tables.
- **Operational complexity.** Compaction, repair, topology changes all require expertise.

**Real-world example:** Discord stores billions of messages in ScyllaDB (Cassandra-compatible, written in C++). Their partition key is `(channel_id, bucket)` where bucket is a time window. This lets them efficiently query "recent messages in channel X" while distributing data evenly across nodes.

---

## Graph Databases

**Examples:** Neo4j, Amazon Neptune, ArangoDB (multi-model), Dgraph

Graph databases model data as nodes and edges (relationships). They excel when the relationships between entities are the primary thing you query.

### When graphs win

- **Social networks:** "Friends of friends who also like X"
- **Recommendation engines:** "Users who bought this also bought..."
- **Fraud detection:** "Find all accounts connected to this flagged account within 3 hops"
- **Knowledge graphs:** Modeling interconnected concepts with typed relationships

### When graphs are overkill

If your queries are "get user by ID" or "list orders for user," a relational database with JOINs handles this trivially. Graph databases add operational complexity for a small ecosystem payoff. Only choose a graph database when relationship traversal is your core query pattern.

---

## Search Engines

**Examples:** Elasticsearch, OpenSearch, Meilisearch, Typesense

Search engines are not primary databases. They are specialized indexes optimized for full-text search, fuzzy matching, faceted filtering, and relevance ranking.

### Common architecture

Your primary database (PostgreSQL) is the source of truth. Changes are replicated to Elasticsearch via a change data capture pipeline (Debezium, custom triggers, or application-level dual writes). Search queries hit Elasticsearch; everything else hits PostgreSQL.

### When to add a search engine

- Full-text search across many fields with relevance ranking
- Autocomplete / typeahead suggestions
- Faceted navigation (filter by category, price range, brand)
- Log aggregation and analysis (the ELK stack)

### Pitfall: dual-write inconsistency

If your application writes to PostgreSQL and Elasticsearch separately, they can drift out of sync when one write succeeds and the other fails. Use a transactional outbox pattern or CDC pipeline to keep them consistent.

---

## Decision Framework

When choosing a database, ask these questions in order:

1. **Is my data relational?** (entities reference each other, need joins, need transactions) -> PostgreSQL
2. **Do I need sub-millisecond reads for simple key lookups?** -> Redis (as a cache, not primary store)
3. **Is my schema truly unpredictable and varies per record?** -> MongoDB / document store
4. **Do I need massive write throughput (millions/sec) across regions?** -> Cassandra / ScyllaDB
5. **Are relationship traversals my core query pattern?** -> Neo4j / graph database
6. **Do I need full-text search with relevance ranking?** -> Elasticsearch / Meilisearch (alongside a primary DB)

**The most common correct answer is PostgreSQL.** It handles relational data, JSONB, full-text search (with `tsvector`), and scales to hundreds of millions of rows on a single node with proper indexing. Start there and add specialized databases only when PostgreSQL demonstrably cannot meet a specific requirement.

---

## Common Mistakes

- **Choosing MongoDB because "SQL is old."** SQL is not old; it is battle-tested. PostgreSQL receives active development and handles most workloads better than people expect.
- **Polyglot persistence on day one.** Running PostgreSQL + Redis + Elasticsearch + MongoDB from the start creates enormous operational burden. Start with one database and add others when you hit a concrete wall.
- **Ignoring operational cost.** Cassandra can handle massive scale, but it requires a team that understands compaction, repair, and data modeling. A managed PostgreSQL instance on RDS costs $200/month and requires zero operational expertise.
- **Confusing marketing with engineering.** Every database vendor claims to be the best at everything. Evaluate based on your specific access patterns, consistency requirements, and team expertise.
