# 14 - Database Engineering

## Diagrams

![Database Types](diagrams/database-types.png)

![Migration Workflow](diagrams/migration-workflow.png)


## Concepts

### Choosing the Right Database

There is no universally "best" database. The right choice depends on your data model, access patterns, consistency requirements, and scale.

| Type | Examples | Best for | Trade-offs |
|------|----------|----------|------------|
| **Relational (SQL)** | PostgreSQL, MySQL | Structured data, complex queries, ACID transactions | Schema rigidity, horizontal scaling is complex |
| **Document** | MongoDB, CouchDB | Semi-structured data, flexible schemas, rapid iteration | Weaker consistency, no joins (typically) |
| **Key-Value** | Redis, DynamoDB | Caching, session storage, high-throughput simple lookups | Limited query capabilities |
| **Column-Family** | Cassandra, ScyllaDB | Time-series, write-heavy workloads, massive scale | Complex data modeling, eventual consistency |
| **Graph** | Neo4j, Amazon Neptune | Relationship-heavy data (social networks, recommendations) | Niche use case, smaller ecosystem |
| **Search** | Elasticsearch, Meilisearch | Full-text search, fuzzy matching, faceted search | Not a primary database, eventual consistency |

### Schema Design & Migrations

**Schema design principles:**
- Design for your read patterns, not just your write patterns
- Normalize to avoid data anomalies, denormalize when performance demands it
- Use appropriate data types (don't store dates as strings, use UUIDs for distributed IDs)
- Add constraints (NOT NULL, UNIQUE, CHECK, FOREIGN KEY) — let the database enforce correctness

**Migrations:**

Migrations are versioned, ordered changes to your database schema. They enable:
- Reproducible database setup (from scratch or from any version)
- Team collaboration (everyone applies the same changes)
- Rollback capability (reverse a migration if something goes wrong)

```sql
-- migrations/001_create_users.sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       TEXT NOT NULL UNIQUE,
    name        TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users (email);

-- migrations/002_add_user_status.sql
ALTER TABLE users ADD COLUMN status TEXT NOT NULL DEFAULT 'active';
CREATE INDEX idx_users_status ON users (status);
```

**Migration tools in Rust:**
- `sqlx migrate` — Built into sqlx, SQL-based migrations
- `diesel` migrations — Diesel ORM's migration system
- `refinery` — Standalone migration runner

**Zero-downtime migrations:**

In production, you can't lock a table for 10 minutes while adding a column. Safe migration patterns:

| Operation | Safe approach |
|-----------|--------------|
| Add column | Add as nullable or with default (instant in PostgreSQL 11+) |
| Remove column | Stop reading it first, deploy, then drop it |
| Rename column | Add new column → backfill → update code → drop old column |
| Add index | `CREATE INDEX CONCURRENTLY` (PostgreSQL) |
| Change column type | Add new column → backfill → update code → drop old column |

### SQL Deep Dive

#### Joins

```sql
-- INNER JOIN: only matching rows from both tables
SELECT o.id, u.name, o.total
FROM orders o
INNER JOIN users u ON o.user_id = u.id;

-- LEFT JOIN: all rows from left table, matching from right (NULL if no match)
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.name;

-- This shows users with 0 orders too (order_count = 0)
```

#### Window Functions

Window functions perform calculations across a set of rows related to the current row — without collapsing them into a single output row.

```sql
-- Rank users by total spending
SELECT
    user_id,
    total_spent,
    RANK() OVER (ORDER BY total_spent DESC) AS spending_rank
FROM user_stats;

-- Running total of daily revenue
SELECT
    date,
    revenue,
    SUM(revenue) OVER (ORDER BY date) AS cumulative_revenue
FROM daily_revenue;

-- Each user's most recent order
SELECT * FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) sub
WHERE rn = 1;
```

#### Common Table Expressions (CTEs)

CTEs make complex queries readable by breaking them into named steps.

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(total) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE_TRUNC('month', created_at)
),
revenue_growth AS (
    SELECT
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
        revenue - LAG(revenue) OVER (ORDER BY month) AS growth
    FROM monthly_revenue
)
SELECT
    month,
    revenue,
    prev_month_revenue,
    growth,
    ROUND(growth / NULLIF(prev_month_revenue, 0) * 100, 1) AS growth_percent
FROM revenue_growth
ORDER BY month;
```

### Connection Pooling

Opening a database connection is expensive (TCP handshake, authentication, TLS). Connection pooling maintains a pool of reusable connections.

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(20)        // Max connections in the pool
    .min_connections(5)         // Keep at least 5 alive
    .acquire_timeout(Duration::from_secs(3))  // Wait max 3s for a connection
    .idle_timeout(Duration::from_secs(300))   // Close idle connections after 5 min
    .connect("postgres://user:pass@localhost/mydb")
    .await?;

// Connections are borrowed and returned automatically
let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
    .bind(user_id)
    .fetch_optional(&pool)
    .await?;
```

**Sizing the pool:**
- Too few connections → requests queue waiting for connections
- Too many connections → database overwhelmed, context switching overhead
- Rule of thumb: `connections = (2 * CPU cores) + number_of_disks` for the database server
- Monitor: connection wait time, active connections, idle connections

### ORMs vs Raw SQL

| Approach | Pros | Cons |
|----------|------|------|
| **Raw SQL (sqlx)** | Full SQL power, compile-time checking, no abstraction leaks | More verbose, manual mapping |
| **Query builder (sea-query)** | Composable queries, type-safe, no ORM overhead | Less readable than raw SQL for complex queries |
| **ORM (diesel)** | Rapid development, type-safe schema, automatic migrations | Learning curve, fights with complex queries, N+1 risk |

**sqlx (compile-time checked SQL):**

```rust
// Compile-time verified: if the query is wrong, it won't compile
let orders = sqlx::query_as!(
    Order,
    r#"
    SELECT id, user_id, total, status as "status: OrderStatus", created_at
    FROM orders
    WHERE user_id = $1 AND status = $2
    ORDER BY created_at DESC
    LIMIT $3
    "#,
    user_id,
    OrderStatus::Completed as _,
    limit
)
.fetch_all(&pool)
.await?;
```

**diesel (ORM):**

```rust
use diesel::prelude::*;

let results = orders::table
    .filter(orders::user_id.eq(user_id))
    .filter(orders::status.eq(OrderStatus::Completed))
    .order(orders::created_at.desc())
    .limit(limit)
    .load::<Order>(&mut conn)?;
```

### NoSQL Data Modeling

NoSQL databases require different thinking than relational databases. Instead of normalizing data, you model for your access patterns.

**DynamoDB single-table design:**

Instead of multiple tables with joins, store all entities in one table with carefully designed partition and sort keys.

```
PK              | SK              | Data
USER#123        | PROFILE         | { name: "Alice", email: "..." }
USER#123        | ORDER#456       | { total: 99.99, status: "shipped" }
USER#123        | ORDER#789       | { total: 149.50, status: "pending" }
ORDER#456       | ITEM#1          | { product: "Widget", qty: 2 }
ORDER#456       | ITEM#2          | { product: "Gadget", qty: 1 }
```

**Access patterns this supports:**
- Get user profile: `PK = USER#123, SK = PROFILE`
- Get all orders for user: `PK = USER#123, SK begins_with ORDER#`
- Get all items in an order: `PK = ORDER#456, SK begins_with ITEM#`

**When to use NoSQL:**
- Simple access patterns (key-value lookups, single-table queries)
- Massive write throughput requirements
- Horizontally scalable workloads
- Schema flexibility for evolving data models

**When NOT to use NoSQL:**
- Complex queries with joins and aggregations
- Strong consistency requirements across multiple entities
- Ad-hoc reporting and analytics
- Small-to-medium scale where PostgreSQL handles the load fine

### Query Optimization

**EXPLAIN ANALYZE** shows how PostgreSQL executes a query and where time is spent:

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.name
ORDER BY order_count DESC
LIMIT 10;
```

**Common performance problems and fixes:**

| Problem | Symptom | Fix |
|---------|---------|-----|
| Missing index | Sequential scan on large table | Add an index on the filtered/joined column |
| N+1 queries | Many small queries in a loop | Use JOINs or batch queries |
| Over-fetching | `SELECT *` when you need 2 columns | Select only needed columns |
| Large result sets | Returning 100K rows to the app | Add LIMIT, paginate, or aggregate in SQL |
| Lock contention | Slow writes, deadlocks | Reduce transaction duration, use advisory locks |
| Bloated indexes | Slow inserts, large disk usage | REINDEX, consider partial indexes |

**Index types in PostgreSQL:**

| Type | Use case |
|------|----------|
| **B-tree** (default) | Equality, range queries (`=`, `<`, `>`, `BETWEEN`) |
| **Hash** | Equality only (`=`), slightly faster than B-tree for this |
| **GIN** | Full-text search, JSONB, array containment |
| **GiST** | Geometric data, range types, full-text search |
| **BRIN** | Very large tables with naturally ordered data (timestamps) |

```sql
-- Partial index: only index active users (smaller, faster)
CREATE INDEX idx_users_active_email ON users (email) WHERE status = 'active';

-- Composite index: for queries filtering on both columns
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

-- GIN index for JSONB
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);
```

## Business Value

- **Preventing costly migrations**: Getting the database choice and schema design right avoids expensive migrations. Migrating from MongoDB to PostgreSQL (or vice versa) can take months and introduce bugs.
- **Performance without hardware**: Proper indexing and query optimization can deliver 10-100x performance improvements at zero infrastructure cost.
- **Zero-downtime deployments**: Safe migration practices enable continuous deployment without database downtime — critical for 24/7 businesses.
- **Reduced infrastructure costs**: Optimized queries reduce database CPU and I/O, directly lowering cloud bills. A single missing index can be the difference between a $200/month and $2,000/month database instance.
- **Developer productivity**: Good schema design and appropriate tooling (sqlx's compile-time checks) prevent runtime errors and reduce debugging time.

## Real-World Examples

### Instagram's PostgreSQL at Scale
Instagram chose PostgreSQL early and scaled it to billions of rows. Key strategies: sharding by user ID, extensive use of partial indexes, connection pooling with PgBouncer, and denormalization for read-heavy access patterns. They demonstrated that PostgreSQL can scale far beyond what most companies need — the database choice matters less than how you use it.

### Discord's Migration from MongoDB to Cassandra to ScyllaDB
Discord started with MongoDB (fast prototyping), migrated to Cassandra when message storage hit scale limits (billions of messages), and then migrated again to ScyllaDB (a Cassandra-compatible database written in C++) for better performance. Each migration was driven by specific access pattern requirements — not technology trends. Their engineering blog documents the trade-offs at each stage.

### Shopify's Schema Migration Strategy
Shopify performs hundreds of database migrations per week across one of the largest MySQL deployments in the world. They built `gh-ost` (GitHub Online Schema Migration) — a tool that performs schema changes without locking tables. This enables zero-downtime deployments even for schema changes that would traditionally lock tables for hours.

### How Figma Scaled PostgreSQL
Figma scaled PostgreSQL to handle their real-time collaborative design tool. Key decisions: horizontal sharding by document ID, PgBouncer for connection pooling, read replicas for analytics queries, and aggressive query optimization. They chose to invest in scaling PostgreSQL rather than migrating to a different database — because their team's PostgreSQL expertise was a competitive advantage.

## Common Mistakes & Pitfalls

- **Premature NoSQL adoption** — Choosing MongoDB because "SQL is old" when your data is highly relational. PostgreSQL handles most workloads better than people expect.

- **No indexes on foreign keys** — Foreign key columns used in JOINs and WHERE clauses without indexes cause full table scans. Always index foreign keys.

- **N+1 queries** — Fetching a list of orders, then querying for each order's items separately. Use JOINs or batch queries (`WHERE id = ANY($1)`).

- **Not using migrations** — Applying schema changes manually in production. One missed step breaks the database. Always use versioned, automated migrations.

- **Ignoring connection pool limits** — Setting max connections to 100 when the database can only handle 50. Monitor and tune pool sizes based on actual database capacity.

- **SELECT \* everywhere** — Fetching all columns when you need two. This wastes network bandwidth, memory, and prevents index-only scans.

- **Treating the database as a message queue** — Polling a table for new rows. Use an actual message queue (Kafka, RabbitMQ, Redis Streams) for pub/sub patterns.

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **PostgreSQL for everything** | One technology to master, rich features | May not be optimal for every access pattern |
| **Polyglot persistence** | Best tool for each job | Operational complexity, data consistency challenges |
| **Heavy normalization** | No data anomalies, storage efficiency | More JOINs, potentially slower reads |
| **Denormalization** | Faster reads, simpler queries | Data duplication, update anomalies |
| **ORM** | Rapid development, type safety | Hides SQL complexity, N+1 risk, learning curve |
| **Raw SQL** | Full control, performance | More verbose, less portable |

## When to Use / When Not to Use

**PostgreSQL — use for:**
- Most applications (it's the safe default)
- Complex queries, reporting, analytics
- ACID transaction requirements
- JSONB for semi-structured data alongside relational data

**Redis — use for:**
- Caching (session storage, query results, computed values)
- Rate limiting, leaderboards, pub/sub
- NOT as a primary database (data loss risk on restart)

**Document DB (MongoDB) — use for:**
- Truly schema-less data where structure varies per record
- Rapid prototyping where schema is unknown
- When you don't need JOINs or complex queries

**Cassandra/ScyllaDB — use for:**
- Write-heavy workloads at massive scale (billions of rows)
- Time-series data, IoT, event logging
- When eventual consistency is acceptable

## Key Takeaways

1. PostgreSQL is the right default for most applications. It handles relational data, JSON, full-text search, and scales further than most companies need.
2. Design schemas for your read patterns. Write patterns are usually simpler than read patterns.
3. Always use database migrations. Manual schema changes in production are a recipe for disaster.
4. Index foreign keys and columns used in WHERE/JOIN/ORDER BY clauses. A missing index is the most common performance problem.
5. Use EXPLAIN ANALYZE to understand query performance. Don't guess — measure.
6. Connection pooling is mandatory for production. Size the pool based on database capacity, not application demand.
7. Choose between ORM and raw SQL based on your team and complexity. sqlx's compile-time checking gives you the best of both worlds in Rust.

## Further Reading

- **Books:**
  - *Designing Data-Intensive Applications* — Martin Kleppmann (2017) — The definitive guide to database concepts and trade-offs
  - *PostgreSQL: Up and Running* — Regina Obe & Leo Hsu (3rd edition) — Practical PostgreSQL guide
  - *The Art of PostgreSQL* — Dimitri Fontaine (2020) — Advanced SQL techniques

- **Papers & Articles:**
  - [Use The Index, Luke](https://use-the-index-luke.com/) — Comprehensive guide to database indexing
  - [Figma's PostgreSQL Scaling](https://www.figma.com/blog/how-figma-scaled-to-multiple-databases/) — Real-world PostgreSQL scaling
  - [Discord's Database Migrations](https://discord.com/blog/how-discord-stores-billions-of-messages) — Migration journey through three databases

- **Crates:**
  - [sqlx](https://crates.io/crates/sqlx) — Async SQL with compile-time checking
  - [diesel](https://crates.io/crates/diesel) — Type-safe ORM and query builder
  - [sea-orm](https://crates.io/crates/sea-orm) — Async ORM built on sqlx
  - [deadpool-postgres](https://crates.io/crates/deadpool-postgres) — Connection pooling
