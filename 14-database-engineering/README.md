# Database Engineering

## Sub-documents

1. [Choosing the Right Database](01-choosing-databases.md) — Relational, document, key-value, column-family, graph, and search databases. Decision framework for when to use each.
2. [Schema Design & Migrations](02-schema-design-and-migrations.md) — Normalization, denormalization, constraints, migration tools in Rust, and zero-downtime migration patterns.
3. [SQL Deep Dive](03-sql-deep-dive.md) — Joins, window functions, CTEs, subqueries, and advanced SQL techniques with practical examples.
4. [Query Optimization](04-query-optimization.md) — EXPLAIN ANALYZE, indexing strategies (B-tree, GIN, partial, covering), the N+1 problem, and connection pooling.
5. [NoSQL Data Modeling](05-nosql-data-modeling.md) — DynamoDB single-table design, Redis patterns (caching, pub/sub, sorted sets), MongoDB document modeling. Discord's migration from Mongo to Cassandra to ScyllaDB.
6. [ORMs vs Raw SQL](06-orms-vs-raw-sql.md) — sqlx, diesel, and sea-orm compared with identical queries. Trade-offs, N+1 problem, and when to use which.

## Diagrams

![Database Types](diagrams/database-types.png)

![Migration Workflow](diagrams/migration-workflow.png)

## Key Takeaways

1. PostgreSQL is the right default for most applications. It handles relational data, JSON, full-text search, and scales further than most companies need.
2. Design schemas for your read patterns. Write patterns are usually simpler than read patterns.
3. Always use database migrations. Manual schema changes in production are a recipe for disaster.
4. Index foreign keys and columns used in WHERE/JOIN/ORDER BY clauses. A missing index is the most common performance problem.
5. Use EXPLAIN ANALYZE to understand query performance. Don't guess — measure.
6. Connection pooling is mandatory for production. Size the pool based on database capacity, not application demand.
7. Choose between ORM and raw SQL based on your team and complexity. sqlx's compile-time checking gives you the best of both worlds in Rust.
8. Model NoSQL tables around access patterns, not entities. List every query before designing the schema.
9. Use Redis as a cache and coordination layer, never as a primary database.

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
