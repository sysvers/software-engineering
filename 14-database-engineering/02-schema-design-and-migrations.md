# Schema Design and Migrations

Good schema design prevents problems that no amount of application code can fix. Bad schema design causes cascading bugs, slow queries, and painful migrations that take months. This document covers how to design schemas correctly and how to evolve them safely in production.

---

## Schema Design Principles

### Design for your read patterns

Your application reads data far more than it writes it. Design tables and indexes around the queries your application actually runs, not around an abstract data model.

**Example:** An e-commerce platform reads "user's recent orders with item details" on every page load. If you normalize orders, order_items, and products into three tables, every page load requires a three-way JOIN. If reads dominate, consider denormalizing the product name and image URL into the order_items table.

### Normalization: the starting point

Normalization eliminates data duplication and update anomalies. Start normalized and denormalize only when you have measured evidence that JOINs are a bottleneck.

| Normal Form | Rule | Example violation |
|-------------|------|-------------------|
| **1NF** | No repeating groups, atomic values | Storing `tags` as a comma-separated string |
| **2NF** | No partial dependencies on composite keys | Order item storing the product name (depends on product_id, not the full composite key) |
| **3NF** | No transitive dependencies | User table storing `city` and `state` when `state` depends on `city_zip` |

**Practical rule:** For most applications, aim for 3NF and denormalize specific tables when query performance demands it.

### Use appropriate data types

```sql
-- BAD: stringly-typed data
CREATE TABLE events (
    id          TEXT,           -- should be UUID or BIGSERIAL
    event_date  TEXT,           -- should be TIMESTAMPTZ
    amount      TEXT,           -- should be NUMERIC or BIGINT (cents)
    is_active   TEXT            -- should be BOOLEAN
);

-- GOOD: proper types with constraints
CREATE TABLE events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_date  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    amount_cents BIGINT NOT NULL CHECK (amount_cents >= 0),
    is_active   BOOLEAN NOT NULL DEFAULT true
);
```

**Why this matters:**
- Proper types enable indexing (you cannot do range queries on stringified dates)
- Constraints catch bugs at the database level before bad data spreads
- Storage is more efficient (a BOOLEAN is 1 byte; TEXT `"true"` is 5 bytes plus overhead)

### Use constraints aggressively

```sql
CREATE TABLE subscriptions (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    plan        TEXT NOT NULL CHECK (plan IN ('free', 'pro', 'enterprise')),
    starts_at   TIMESTAMPTZ NOT NULL,
    ends_at     TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Business rule: end date must be after start date
    CONSTRAINT valid_date_range CHECK (ends_at IS NULL OR ends_at > starts_at),

    -- One active subscription per user
    CONSTRAINT unique_active_subscription
        UNIQUE (user_id) WHERE (ends_at IS NULL OR ends_at > NOW())
        -- Note: this requires a partial unique index, shown below
);

-- Partial unique index enforcing one active subscription per user
CREATE UNIQUE INDEX idx_one_active_sub_per_user
    ON subscriptions (user_id)
    WHERE ends_at IS NULL OR ends_at > NOW();
```

### UUIDs vs auto-increment IDs

| Approach | Pros | Cons |
|----------|------|------|
| **BIGSERIAL** | Compact (8 bytes), fast inserts, natural ordering | Exposes count, not safe for distributed systems, enumerable |
| **UUIDv4** | Globally unique, safe to expose in URLs | 16 bytes, random order hurts B-tree locality |
| **UUIDv7** | Globally unique AND time-ordered | 16 bytes (but good B-tree locality) |
| **ULID** | Like UUIDv7, sortable, Crockford base32 | Less native support than UUID |

**Recommendation:** Use `BIGSERIAL` for internal IDs (primary keys, foreign keys). Use `UUIDv7` for public-facing identifiers (API responses, URLs). Store both on the same row if needed.

---

## Migrations

Migrations are versioned, ordered SQL scripts that evolve your database schema. They are non-negotiable for any production system.

### Why migrations matter

- **Reproducibility:** Any developer can recreate the database from scratch by running all migrations in order.
- **Collaboration:** Multiple developers can add migrations in parallel without conflicts (as long as they don't modify the same objects).
- **Auditability:** The migration history is a changelog of your schema.
- **Rollback:** If a migration breaks something, you can reverse it (if you wrote a down migration).

### Migration structure

```
migrations/
  001_create_users.sql
  002_create_orders.sql
  003_add_user_status.sql
  004_create_order_items.sql
  005_add_indexes_for_reporting.sql
```

Each file is applied exactly once, in order. A metadata table (`_sqlx_migrations`, `schema_migrations`, etc.) tracks which migrations have been applied.

### Migration tools in Rust

#### sqlx migrate

Built into sqlx. Runs plain SQL files. Supports reversible migrations (up/down pairs) and simple forward-only migrations.

```bash
# Create a new migration
sqlx migrate add create_users

# Run pending migrations
sqlx migrate run

# Revert the last migration
sqlx migrate revert
```

```text
// Run migrations at application startup (embedded in binary)
AWAIT RUN_MIGRATIONS("./migrations", pool)
```

The `sqlx::migrate!` macro embeds migration files into the binary at compile time. No need to ship migration files alongside the binary in production.

#### Diesel migrations

Diesel generates Rust code from your schema, keeping your models and database in sync at compile time.

```bash
# Create a migration
diesel migration generate create_users

# Run migrations
diesel migration run

# Revert the last migration
diesel migration revert

# Regenerate schema.rs from the database
diesel print-schema > src/schema.rs
```

```text
// migrations/2024-01-15-000001_create_users/up.sql
// (SQL content -- see the SQL migration files)
// Creates users table with id, email, name, created_at columns

// migrations/2024-01-15-000001_create_users/down.sql
// Drops the users table
```

#### Refinery

A standalone migration runner that works with any database library. Supports SQL and Rust-based migrations.

```text
// Embed migrations from "./migrations" directory

PROCEDURE RUN_MIGRATIONS(db_url):
    config ← CONFIG_FROM_DB_URI(db_url)
    AWAIT EMBEDDED_MIGRATIONS.RUNNER().RUN_ASYNC(config)
```

---

## Zero-Downtime Migrations

In production, you cannot lock a table for minutes while restructuring it. Users are making requests. Every migration must be backward-compatible with the currently deployed application code.

### The expand-contract pattern

Every breaking schema change follows three phases:

1. **Expand:** Add the new structure alongside the old one.
2. **Migrate code:** Deploy application code that uses the new structure.
3. **Contract:** Remove the old structure.

### Safe operations reference

| Operation | Safe approach | Why |
|-----------|--------------|-----|
| **Add column** | Add as `NULL` or with a `DEFAULT` (instant in PG 11+) | No table rewrite needed |
| **Remove column** | Stop reading it -> deploy -> drop it | Dropping while code reads it causes errors |
| **Rename column** | Add new column -> dual-write -> backfill -> switch reads -> drop old | Direct rename breaks running code |
| **Add index** | `CREATE INDEX CONCURRENTLY` | Regular `CREATE INDEX` locks the table |
| **Change column type** | Add new column -> backfill -> switch -> drop old | `ALTER COLUMN TYPE` rewrites the table |
| **Add NOT NULL** | Add CHECK constraint as `NOT VALID` -> validate separately | Avoids full table scan under lock |

### Real example: adding NOT NULL safely

```sql
-- Step 1: Add the column as nullable (instant, no lock)
ALTER TABLE users ADD COLUMN phone TEXT;

-- Step 2: Deploy code that always writes phone on new inserts
-- Step 3: Backfill existing rows
UPDATE users SET phone = 'unknown' WHERE phone IS NULL;
-- (Do this in batches for large tables to avoid long transactions)

-- Step 4: Add the NOT NULL constraint without validating (instant)
ALTER TABLE users ADD CONSTRAINT users_phone_not_null
    CHECK (phone IS NOT NULL) NOT VALID;

-- Step 5: Validate the constraint (scans the table but doesn't lock writes)
ALTER TABLE users VALIDATE CONSTRAINT users_phone_not_null;
```

### Backfilling large tables in batches

Never `UPDATE users SET new_col = computed_value` on a 100-million-row table. It creates a massive transaction that locks the table and fills the WAL.

```text
PROCEDURE BACKFILL_IN_BATCHES(pool):
    batch_size ← 10000
    last_id ← 0

    LOOP:
        result ← AWAIT EXECUTE pool:
            "UPDATE users
             SET phone = 'unknown'
             WHERE id > last_id AND id ≤ last_id + batch_size AND phone IS NULL"

        IF result.rows_affected = 0 THEN
            // Check if we've passed the max ID
            max_id ← AWAIT QUERY pool: "SELECT MAX(id) FROM users"

            IF max_id IS present AND last_id + batch_size < max_id THEN
                last_id ← last_id + batch_size
                CONTINUE
            ELSE
                BREAK

        last_id ← last_id + batch_size
        // Optional: sleep to reduce load
        SLEEP 100 milliseconds
```

---

## Common Schema Design Pitfalls

- **No foreign keys.** "We enforce this in application code." Application code has bugs. Databases do not skip constraint checks.
- **EAV (Entity-Attribute-Value) tables.** A generic `key-value` table for all entity attributes. Impossible to query efficiently, no type safety, no constraints. Use JSONB if you need flexibility.
- **Storing money as floats.** `FLOAT` cannot represent $0.10 exactly. Use `BIGINT` (cents) or `NUMERIC(19,4)`.
- **No `created_at` / `updated_at` columns.** You will always wish you had them. Add them to every table from day one.
- **Huge migrations in a single file.** Break large changes into small, incremental migrations. Each should be independently deployable and reversible.
- **Not testing migrations.** Run your full migration suite against a copy of production data before deploying. A migration that works on an empty database may fail on real data.
