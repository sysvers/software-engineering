# Query Optimization

The difference between a 50ms query and a 5-second query is usually one missing index or one poorly structured query. This document covers how to diagnose slow queries, which indexes to create, and how to avoid the most common performance traps.

---

## EXPLAIN ANALYZE

`EXPLAIN ANALYZE` is the single most important tool for understanding query performance. It runs the query and shows the execution plan with actual timings.

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

### Reading the output

```
Limit  (cost=1234.56..1234.58 rows=10 width=40) (actual time=45.2..45.3 rows=10 loops=1)
  -> Sort  (cost=1234.56..1240.12 rows=2222 width=40) (actual time=45.2..45.2 rows=10 loops=1)
        Sort Key: (count(o.id)) DESC
        Sort Method: top-N heapsort  Memory: 25kB
        -> HashAggregate  (cost=1180.00..1202.22 rows=2222 width=40) (actual time=44.5..44.8 rows=2222 loops=1)
              -> Hash Left Join  (cost=100.00..1100.00 rows=16000 width=40) (actual time=5.1..35.2 rows=16000 loops=1)
                    -> Seq Scan on users u  (cost=0.00..80.00 rows=2222 width=36) (actual time=0.01..1.2 rows=2222 loops=1)
                          Filter: (status = 'active')
                          Rows Removed by Filter: 7778
                    -> Hash  (cost=60.00..60.00 rows=16000 width=12) (actual time=4.8..4.8 rows=16000 loops=1)
                          -> Seq Scan on orders o  (cost=0.00..60.00 rows=16000 width=12) (actual time=0.01..2.1 rows=16000 loops=1)
```

**Key things to look for:**

| Signal | What it means | What to do |
|--------|--------------|------------|
| **Seq Scan** on a large table | Full table scan, reading every row | Add an index on the filter/join column |
| **actual rows** >> **rows** (estimate) | Planner has bad statistics | Run `ANALYZE tablename` |
| **Nested Loop** with high loops count | Inner scan runs many times | Consider a hash join (add index or increase work_mem) |
| **Sort Method: external merge Disk** | Sort spilled to disk | Increase `work_mem` or add an index to avoid sorting |
| **Rows Removed by Filter** is very high | Index scan reads many rows but filter discards most | Use a more selective index or partial index |

### The optimization workflow

1. **Identify the slow query.** Use `pg_stat_statements` to find queries by total execution time.
2. **Run EXPLAIN ANALYZE.** Read bottom-up; the innermost operations execute first.
3. **Find the bottleneck.** Look for Seq Scans on large tables, high loop counts, disk sorts.
4. **Add the fix.** Usually an index, sometimes a query rewrite.
5. **Measure again.** Run EXPLAIN ANALYZE after the fix to confirm improvement.

---

## Indexing Strategies

### B-tree indexes (the default)

B-tree is the default index type. It supports equality (`=`), range (`<`, `>`, `BETWEEN`), and ordering (`ORDER BY`).

```sql
-- Single-column index
CREATE INDEX idx_users_email ON users (email);

-- This index helps:
--   WHERE email = 'alice@example.com'
--   WHERE email LIKE 'alice%'  (prefix match only)
--   ORDER BY email
```

### Composite indexes

A composite index covers queries that filter on multiple columns. Column order matters.

```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

-- This index helps:
--   WHERE user_id = 123                     (uses first column)
--   WHERE user_id = 123 AND status = 'paid' (uses both columns)

-- This index does NOT help:
--   WHERE status = 'paid'                   (cannot skip the first column)
```

**Rule of thumb:** Put the most selective column (highest cardinality) first if all columns are used in equality checks. If one column uses a range (`>`, `<`, `BETWEEN`), put it last.

### Partial indexes

A partial index only indexes rows matching a condition. Smaller index, faster lookups.

```sql
-- Only 5% of users are active, and most queries filter for active users
CREATE INDEX idx_users_active ON users (email) WHERE status = 'active';

-- This index is 20x smaller than indexing all users
-- Queries MUST include WHERE status = 'active' to use this index
```

**Real-world example:** An orders table with 10 million rows, but only 50,000 are `status = 'pending'`. A partial index on pending orders is tiny and lightning-fast.

```sql
CREATE INDEX idx_orders_pending ON orders (created_at)
    WHERE status = 'pending';
```

### GIN indexes (for JSONB, arrays, full-text)

GIN (Generalized Inverted Index) is designed for values that contain multiple elements.

```sql
-- Index JSONB documents for containment queries
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- Now this is fast:
SELECT * FROM products WHERE metadata @> '{"color": "red"}';

-- Full-text search index
CREATE INDEX idx_articles_fts ON articles USING GIN (to_tsvector('english', title || ' ' || body));

-- Array containment
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);
SELECT * FROM posts WHERE tags @> ARRAY['rust', 'database'];
```

### Covering indexes (INCLUDE)

A covering index includes extra columns so the query can be answered entirely from the index, without touching the table (an "index-only scan").

```sql
-- The query needs user_id, email, and name for active users
CREATE INDEX idx_users_active_covering ON users (status)
    INCLUDE (email, name)
    WHERE status = 'active';

-- This query can be satisfied entirely from the index:
SELECT email, name FROM users WHERE status = 'active';
```

### Expression indexes

Index the result of a function or expression.

```sql
-- Index lowercased emails for case-insensitive lookups
CREATE INDEX idx_users_email_lower ON users (LOWER(email));

-- Now this uses the index:
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- Index the date portion of a timestamp
CREATE INDEX idx_orders_date ON orders (DATE(created_at));
```

### Index maintenance

Indexes speed up reads but slow down writes. Every INSERT and UPDATE must also update every index on the table.

**Signs of over-indexing:**
- Slow inserts/updates
- Large disk usage relative to table size
- Many indexes that `pg_stat_user_indexes` shows are never scanned

```sql
-- Find unused indexes
SELECT
    schemaname, tablename, indexrelname,
    idx_scan,  -- number of times this index was used
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## The N+1 Query Problem

The N+1 problem occurs when you fetch a list of N items, then execute a separate query for each item's related data.

### The problem

```rust
// BAD: N+1 queries
let users = sqlx::query_as::<_, User>("SELECT * FROM users LIMIT 50")
    .fetch_all(&pool).await?;

for user in &users {
    // This executes 50 separate queries!
    let orders = sqlx::query_as::<_, Order>(
        "SELECT * FROM orders WHERE user_id = $1"
    )
    .bind(user.id)
    .fetch_all(&pool).await?;

    // process user + orders...
}
// Total: 1 + 50 = 51 queries
```

### The fix: batch loading

```rust
// GOOD: 2 queries total
let users = sqlx::query_as::<_, User>("SELECT * FROM users LIMIT 50")
    .fetch_all(&pool).await?;

let user_ids: Vec<i64> = users.iter().map(|u| u.id).collect();

let orders = sqlx::query_as::<_, Order>(
    "SELECT * FROM orders WHERE user_id = ANY($1)"
)
.bind(&user_ids)
.fetch_all(&pool).await?;

// Group orders by user_id in application code
let orders_by_user: HashMap<i64, Vec<&Order>> = orders
    .iter()
    .fold(HashMap::new(), |mut acc, order| {
        acc.entry(order.user_id).or_default().push(order);
        acc
    });
// Total: 2 queries regardless of N
```

### The fix: JOIN

```rust
// GOOD: single query with JOIN
let results = sqlx::query_as::<_, UserWithOrder>(
    "SELECT u.id, u.name, o.id AS order_id, o.total, o.status
     FROM users u
     LEFT JOIN orders o ON u.id = o.user_id
     ORDER BY u.id, o.created_at DESC"
)
.fetch_all(&pool).await?;
// Total: 1 query
```

---

## Connection Pooling

Opening a database connection is expensive: TCP handshake, TLS negotiation, authentication, memory allocation on the server side. Connection pooling reuses connections across requests.

### Configuring a pool in Rust

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

let pool = PgPoolOptions::new()
    .max_connections(20)
    .min_connections(5)
    .acquire_timeout(Duration::from_secs(3))
    .idle_timeout(Duration::from_secs(300))
    .max_lifetime(Duration::from_secs(1800))
    .connect("postgres://user:pass@localhost/mydb")
    .await?;
```

### Sizing the pool

**Too few connections:** Requests queue waiting for an available connection. You see high `acquire_timeout` errors and increased latency under load.

**Too many connections:** The database server is overwhelmed. PostgreSQL creates a separate process per connection, each consuming memory and CPU for context switching. A database that handles 50 concurrent queries well may collapse at 500.

**The formula:**
```
optimal_connections = (2 * cpu_cores) + number_of_spindles
```

For a typical cloud database with 4 vCPUs and SSD storage:
```
optimal_connections = (2 * 4) + 1 = 9
```

If you have 10 application servers each with a pool of 20, that is 200 connections to the database. Use PgBouncer as an external connection pooler to multiplex application connections onto a smaller number of database connections.

### Monitoring connection health

```sql
-- Current connection count by state
SELECT state, COUNT(*)
FROM pg_stat_activity
GROUP BY state;

-- Long-running queries (potential connection hogs)
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > INTERVAL '5 minutes';
```

---

## Real Performance Improvement Stories

### Missing index on a foreign key

**Symptom:** The `GET /users/:id/orders` endpoint takes 3 seconds on a table with 5 million orders.

**Diagnosis:**
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;
-- Seq Scan on orders  (actual time=0.05..2850.00 rows=12 loops=1)
--   Filter: (user_id = 123)
--   Rows Removed by Filter: 4999988
```

**Fix:**
```sql
CREATE INDEX idx_orders_user_id ON orders (user_id);
```

**Result:** Query time drops from 2850ms to 0.3ms. The index lookup goes directly to the 12 matching rows instead of scanning 5 million.

### N+1 in a dashboard

**Symptom:** Admin dashboard takes 8 seconds to load. The page shows 100 users with their latest order.

**Diagnosis:** Application log shows 101 queries (1 for users, 100 for orders).

**Fix:** Replace the loop with a single query using `LATERAL JOIN`:
```sql
SELECT u.name, u.email, latest_order.*
FROM users u
CROSS JOIN LATERAL (
    SELECT o.total, o.status, o.created_at
    FROM orders o WHERE o.user_id = u.id
    ORDER BY o.created_at DESC LIMIT 1
) latest_order
ORDER BY u.name LIMIT 100;
```

**Result:** Dashboard loads in 50ms.

### Partial index for status filtering

**Symptom:** Background job that processes pending orders runs every minute and takes 30 seconds to find pending orders in a 50-million-row table.

**Diagnosis:** Full B-tree index on `status` is 1.2 GB and includes all statuses.

**Fix:**
```sql
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';
```

**Result:** Index is 2 MB instead of 1.2 GB. Query time drops from 30 seconds to 5ms.

---

## Pitfalls

- **Not running ANALYZE after bulk loads.** PostgreSQL's query planner relies on table statistics. After inserting millions of rows, run `ANALYZE tablename` so the planner makes good decisions.
- **Indexing every column.** Each index slows down writes and consumes disk. Only index columns that appear in WHERE, JOIN, and ORDER BY clauses of actual queries.
- **Ignoring `work_mem`.** Sorts and hash joins that exceed `work_mem` spill to disk and become dramatically slower. Monitor `Sort Method: external merge` in EXPLAIN output.
- **Using OFFSET for pagination.** `OFFSET 100000` still reads and discards 100,000 rows. Use keyset pagination instead: `WHERE id > $last_seen_id ORDER BY id LIMIT 20`.
- **Not using prepared statements.** Parsing and planning the same query thousands of times per second wastes CPU. Connection pools and ORMs typically handle this, but verify.
- **Premature optimization.** Do not add indexes before you have queries and data. Add them when EXPLAIN ANALYZE shows a problem.
