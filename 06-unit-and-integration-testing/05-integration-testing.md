# Integration Testing

## What Integration Tests Verify

Unit tests verify that individual functions work correctly in isolation. Integration tests verify that components work correctly *together*. They test the seams — the boundaries where your code meets databases, APIs, file systems, and other services.

An integration test answers questions that unit tests cannot:
- Does this SQL query actually return the right data from a real database?
- Does this API endpoint correctly parse a JSON request body and return a proper response?
- Does the message consumer correctly deserialize and process messages from the queue?
- Does the configuration loader handle real TOML files on the actual file system?

## Rust's Integration Test Structure

Rust has a built-in convention for integration tests. Files in the `tests/` directory at the crate root are compiled as separate crates and can only access your library's public API.

```
my_project/
  src/
    lib.rs
    db.rs
    api.rs
  tests/
    test_db.rs        # Integration test: database operations
    test_api.rs       # Integration test: API endpoints
    common/
      mod.rs          # Shared test utilities
```

```text
// tests/test_db
ASYNC TEST create_and_retrieve_user
    pool <- SETUP_TEST_DB()
    repo <- NEW UserRepository(pool)

    created <- repo.CREATE("Alice", "alice@example.com")
    found <- repo.FIND_BY_ID(created.id)

    ASSERT found.name = "Alice"
    ASSERT found.email = "alice@example.com"

    CLEANUP_TEST_DB(pool)
```

Integration tests are run with `cargo test` alongside unit tests. To run only integration tests, use `cargo test --test test_db`.

## Testing with Real Databases

### The Case for Real Databases in Tests

Mocking a database in unit tests is valid for testing business logic. But at some point, you need to verify that your SQL actually works against a real database engine. Query syntax, index behavior, constraint enforcement, transaction semantics — these cannot be mocked accurately.

### sqlx Test Fixtures

The `sqlx` crate provides first-class test support. The `#[sqlx::test]` macro creates a fresh test database for each test, runs migrations, and tears everything down when the test completes.

```toml
[dev-dependencies]
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "migrate"] }
```

```text
// Database test with automatic migration
ASYNC TEST user_email_must_be_unique(pool: PgPool)
    // The pool is connected to a fresh, migrated test database
    EXECUTE SQL "INSERT INTO users (name, email) VALUES ($1, $2)"
        WITH ("Alice", "alice@example.com")
        ON pool

    result <- EXECUTE SQL "INSERT INTO users (name, email) VALUES ($1, $2)"
        WITH ("Bob", "alice@example.com")
        ON pool

    ASSERT result IS error, "Duplicate email should be rejected by unique constraint"
```

Each test gets its own database (or transaction), so tests cannot interfere with each other.

### Fixture Data with sqlx::test

Load seed data for tests that need a populated database:

```text
// Database test with migration and fixtures (users.sql, orders.sql)
ASYNC TEST find_orders_for_user(pool: PgPool)
    orders <- QUERY "SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at"
        WITH (1)
        FETCH ALL FROM pool

    ASSERT orders.length = 3
    ASSERT orders[0].total = 29.99
```

## API Endpoint Testing

Integration tests for HTTP APIs verify the full request-response cycle: routing, middleware, deserialization, business logic, serialization, and status codes.

### Testing with axum

```text
FUNCTION APP() -> Router
    RETURN Router
        .ROUTE("/users", POST -> CREATE_USER)
        .ROUTE("/users/:id", GET -> GET_USER)

ASYNC TEST create_user_returns_201
    app <- APP()
    response <- app.SEND_REQUEST(
        method: "POST",
        uri: "/users",
        header: "Content-Type: application/json",
        body: '{"name": "Alice", "email": "alice@example.com"}'
    )
    ASSERT response.status = 201 CREATED

ASYNC TEST get_nonexistent_user_returns_404
    app <- APP()
    response <- app.SEND_REQUEST(uri: "/users/99999", body: empty)
    ASSERT response.status = 404 NOT FOUND

ASYNC TEST create_user_with_invalid_json_returns_400
    app <- APP()
    response <- app.SEND_REQUEST(
        method: "POST",
        uri: "/users",
        header: "Content-Type: application/json",
        body: '{"name": }'  // malformed JSON
    )
    ASSERT response.status = 400 BAD REQUEST
```

This pattern tests the real router, middleware, and handlers without starting a TCP server. The `oneshot` method sends a single request through the application stack.

## Test Containers

When your tests need real infrastructure (PostgreSQL, Redis, Kafka, Elasticsearch), test containers spin up Docker containers on demand and tear them down when tests complete.

```toml
[dev-dependencies]
testcontainers = "0.23"
testcontainers-modules = { version = "0.11", features = ["postgres"] }
```

```text
ASYNC TEST test_with_real_postgres
    container <- START Postgres container
    port <- container.GET_HOST_PORT(5432)

    connection_string <- "postgres://postgres:postgres@localhost:" + port + "/postgres"
    pool <- CONNECT_TO_PG(connection_string)

    // Run migrations
    RUN_MIGRATIONS("./migrations", pool)

    // Now test with a real PostgreSQL instance
    result <- QUERY "SELECT 1 + 1 AS sum" FETCH ONE FROM pool

    sum <- result.GET("sum")
    ASSERT sum = 2

    // Container is automatically stopped and removed when dropped
```

Test containers are slower than in-memory databases but provide exact parity with production. Use them when you need to test database-specific features (PostgreSQL's JSONB operators, full-text search, or advisory locks).

## Test Isolation Strategies

The hardest part of integration testing is ensuring tests do not interfere with each other. A test that inserts data should not cause another test to see unexpected rows.

### Strategy 1: Transaction Rollback

Wrap each test in a transaction and roll it back at the end. The database never sees the test's writes.

```text
ASYNC TEST test_within_transaction
    pool <- GET_SHARED_POOL()
    tx <- pool.BEGIN_TRANSACTION()

    EXECUTE SQL "INSERT INTO users (name, email) VALUES ($1, $2)"
        WITH ("Test User", "test@example.com") ON tx

    count <- QUERY "SELECT COUNT(*) FROM users WHERE email = $1"
        WITH ("test@example.com") FETCH ONE FROM tx

    ASSERT count = 1

    // Transaction is rolled back when tx is dropped (not committed)
```

Pros: fast, no cleanup needed. Cons: cannot test commit-dependent behavior (triggers that fire on commit, sequences that advance).

### Strategy 2: Database Per Test

Create a fresh database for each test. This is what `#[sqlx::test]` does by default. Each test gets a completely clean database with migrations applied.

Pros: complete isolation, tests commit-dependent behavior. Cons: slower due to database creation overhead.

### Strategy 3: Truncate Tables

Share a database but truncate all tables before each test.

```text
ASYNC FUNCTION RESET_DATABASE(pool)
    EXECUTE SQL "TRUNCATE users, orders, payments RESTART IDENTITY CASCADE" ON pool

ASYNC TEST test_with_clean_tables
    pool <- GET_SHARED_POOL()
    RESET_DATABASE(pool)

    // Test runs against empty tables
```

Pros: faster than creating a database per test. Cons: tests must not run in parallel against the same database unless they use distinct data.

### Strategy 4: Unique Test Data

Use unique identifiers in test data so tests cannot collide:

```text
ASYNC TEST test_with_unique_data
    pool <- GET_SHARED_POOL()
    unique_email <- "test-" + NEW_UUID() + "@example.com"

    EXECUTE SQL "INSERT INTO users (name, email) VALUES ($1, $2)"
        WITH ("Test User", unique_email) ON pool

    user <- QUERY "SELECT * FROM users WHERE email = $1"
        WITH (unique_email) FETCH ONE FROM pool

    ASSERT user.name = "Test User"
```

Pros: tests can run in parallel. Cons: leftover data accumulates (needs periodic cleanup), assertions must filter by unique ID.

## Shared Test Utilities

Avoid duplicating setup code across integration tests. Use a shared module:

```text
// tests/common/mod

ASYNC FUNCTION SETUP_TEST_DB() -> PgPool
    database_url <- ENV("TEST_DATABASE_URL") OR "postgres://localhost/myapp_test"
    pool <- PgPool.CONNECT(database_url)
    RUN_MIGRATIONS("./migrations", pool)
    RETURN pool

FUNCTION TEST_USER(name: string, email: string) -> CreateUserRequest
    RETURN CreateUserRequest { name, email }
```

```text
// tests/test_users
IMPORT common

ASYNC TEST create_user_persists_to_database
    pool <- common.SETUP_TEST_DB()
    // ...
```

## When to Write Integration Tests vs. Unit Tests

| Scenario | Unit Test | Integration Test |
|----------|-----------|-----------------|
| Business rule: discount must not exceed 50% | Yes | |
| SQL query returns correct results | | Yes |
| API returns 400 for invalid input | | Yes |
| Password hashing produces valid hash | Yes | |
| Config file is loaded correctly | | Yes |
| Database constraints enforce uniqueness | | Yes |
| Error message formatting | Yes | |
| Service-to-service HTTP communication | | Yes |

The rule of thumb: if the correctness depends on an external system's behavior, write an integration test. If the logic is self-contained, write a unit test.

## Common Pitfalls

### Slow Integration Test Suites

Integration tests are inherently slower than unit tests. Mitigate this by:
- Running them in parallel when isolation allows it
- Using connection pooling instead of creating new connections per test
- Running integration tests separately from unit tests in CI (`cargo test --lib` for unit, `cargo test --test '*'` for integration)

### Flaky Tests from Shared State

If integration tests pass individually but fail when run together, shared mutable state is the cause. Use one of the isolation strategies above. Never rely on test execution order.

### Testing Too Much at the Integration Level

If your integration tests are testing business logic through an API endpoint, you are doing too much work at the wrong level. Test the business logic in unit tests. Use integration tests only to verify that the wiring is correct — that the API correctly calls the business logic and returns the result.

## Key Takeaways

1. Integration tests verify that components work together. They catch bugs that unit tests cannot: SQL errors, serialization mismatches, misconfigured routes.
2. Use Rust's `tests/` directory for integration tests. They compile as separate crates and test only your public API.
3. Test with real databases when query correctness matters. Use `sqlx::test` for per-test database isolation or test containers for exact production parity.
4. Choose an isolation strategy that matches your needs: transaction rollback for speed, database-per-test for complete isolation, unique data for parallelism.
5. Keep integration tests focused on wiring and boundaries. Push business logic verification into unit tests.
6. Invest in shared test utilities early. Duplicated setup code across integration tests becomes a maintenance burden quickly.
