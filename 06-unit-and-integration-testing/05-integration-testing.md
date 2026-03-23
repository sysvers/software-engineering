# 05 - Integration Testing

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

```rust
// tests/test_db.rs
use my_project::UserRepository;

#[tokio::test]
async fn create_and_retrieve_user() {
    let pool = setup_test_db().await;
    let repo = UserRepository::new(pool.clone());

    let created = repo.create("Alice", "alice@example.com").await.unwrap();
    let found = repo.find_by_id(created.id).await.unwrap().unwrap();

    assert_eq!(found.name, "Alice");
    assert_eq!(found.email, "alice@example.com");

    cleanup_test_db(pool).await;
}
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

```rust
use sqlx::PgPool;

#[sqlx::test(migrations = "./migrations")]
async fn user_email_must_be_unique(pool: PgPool) {
    // The pool is connected to a fresh, migrated test database
    sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind("Alice")
        .bind("alice@example.com")
        .execute(&pool)
        .await
        .unwrap();

    let result = sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind("Bob")
        .bind("alice@example.com")
        .execute(&pool)
        .await;

    assert!(result.is_err(), "Duplicate email should be rejected by unique constraint");
}
```

Each test gets its own database (or transaction), so tests cannot interfere with each other.

### Fixture Data with sqlx::test

Load seed data for tests that need a populated database:

```rust
#[sqlx::test(
    migrations = "./migrations",
    fixtures("users", "orders")  // loads tests/fixtures/users.sql, orders.sql
)]
async fn find_orders_for_user(pool: PgPool) {
    let orders = sqlx::query_as::<_, Order>(
        "SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at"
    )
    .bind(1)
    .fetch_all(&pool)
    .await
    .unwrap();

    assert_eq!(orders.len(), 3);
    assert_eq!(orders[0].total, 29.99);
}
```

## API Endpoint Testing

Integration tests for HTTP APIs verify the full request-response cycle: routing, middleware, deserialization, business logic, serialization, and status codes.

### Testing with axum

```rust
use axum::{Router, body::Body, http::{Request, StatusCode}};
use tower::ServiceExt; // for `oneshot`

fn app() -> Router {
    Router::new()
        .route("/users", post(create_user))
        .route("/users/:id", get(get_user))
}

#[tokio::test]
async fn create_user_returns_201() {
    let app = app();

    let response = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/users")
                .header("Content-Type", "application/json")
                .body(Body::from(r#"{"name": "Alice", "email": "alice@example.com"}"#))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::CREATED);
}

#[tokio::test]
async fn get_nonexistent_user_returns_404() {
    let app = app();

    let response = app
        .oneshot(
            Request::builder()
                .uri("/users/99999")
                .body(Body::empty())
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::NOT_FOUND);
}

#[tokio::test]
async fn create_user_with_invalid_json_returns_400() {
    let app = app();

    let response = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/users")
                .header("Content-Type", "application/json")
                .body(Body::from(r#"{"name": }"#)) // malformed JSON
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::BAD_REQUEST);
}
```

This pattern tests the real router, middleware, and handlers without starting a TCP server. The `oneshot` method sends a single request through the application stack.

## Test Containers

When your tests need real infrastructure (PostgreSQL, Redis, Kafka, Elasticsearch), test containers spin up Docker containers on demand and tear them down when tests complete.

```toml
[dev-dependencies]
testcontainers = "0.23"
testcontainers-modules = { version = "0.11", features = ["postgres"] }
```

```rust
use testcontainers::runners::AsyncRunner;
use testcontainers_modules::postgres::Postgres;

#[tokio::test]
async fn test_with_real_postgres() {
    let container = Postgres::default().start().await.unwrap();
    let port = container.get_host_port_ipv4(5432).await.unwrap();

    let connection_string = format!("postgres://postgres:postgres@localhost:{}/postgres", port);
    let pool = PgPool::connect(&connection_string).await.unwrap();

    // Run migrations
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();

    // Now test with a real PostgreSQL instance
    let result = sqlx::query("SELECT 1 + 1 AS sum")
        .fetch_one(&pool)
        .await
        .unwrap();

    let sum: i32 = result.get("sum");
    assert_eq!(sum, 2);

    // Container is automatically stopped and removed when dropped
}
```

Test containers are slower than in-memory databases but provide exact parity with production. Use them when you need to test database-specific features (PostgreSQL's JSONB operators, full-text search, or advisory locks).

## Test Isolation Strategies

The hardest part of integration testing is ensuring tests do not interfere with each other. A test that inserts data should not cause another test to see unexpected rows.

### Strategy 1: Transaction Rollback

Wrap each test in a transaction and roll it back at the end. The database never sees the test's writes.

```rust
#[tokio::test]
async fn test_within_transaction() {
    let pool = get_shared_pool().await;
    let mut tx = pool.begin().await.unwrap();

    sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind("Test User")
        .bind("test@example.com")
        .execute(&mut *tx)
        .await
        .unwrap();

    let count: (i64,) = sqlx::query_as("SELECT COUNT(*) FROM users WHERE email = $1")
        .bind("test@example.com")
        .fetch_one(&mut *tx)
        .await
        .unwrap();

    assert_eq!(count.0, 1);

    // Transaction is rolled back when `tx` is dropped (not committed)
}
```

Pros: fast, no cleanup needed. Cons: cannot test commit-dependent behavior (triggers that fire on commit, sequences that advance).

### Strategy 2: Database Per Test

Create a fresh database for each test. This is what `#[sqlx::test]` does by default. Each test gets a completely clean database with migrations applied.

Pros: complete isolation, tests commit-dependent behavior. Cons: slower due to database creation overhead.

### Strategy 3: Truncate Tables

Share a database but truncate all tables before each test.

```rust
async fn reset_database(pool: &PgPool) {
    sqlx::query("TRUNCATE users, orders, payments RESTART IDENTITY CASCADE")
        .execute(pool)
        .await
        .unwrap();
}

#[tokio::test]
async fn test_with_clean_tables() {
    let pool = get_shared_pool().await;
    reset_database(&pool).await;

    // Test runs against empty tables
}
```

Pros: faster than creating a database per test. Cons: tests must not run in parallel against the same database unless they use distinct data.

### Strategy 4: Unique Test Data

Use unique identifiers in test data so tests cannot collide:

```rust
#[tokio::test]
async fn test_with_unique_data() {
    let pool = get_shared_pool().await;
    let unique_email = format!("test-{}@example.com", Uuid::new_v4());

    sqlx::query("INSERT INTO users (name, email) VALUES ($1, $2)")
        .bind("Test User")
        .bind(&unique_email)
        .execute(&pool)
        .await
        .unwrap();

    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE email = $1")
        .bind(&unique_email)
        .fetch_one(&pool)
        .await
        .unwrap();

    assert_eq!(user.name, "Test User");
}
```

Pros: tests can run in parallel. Cons: leftover data accumulates (needs periodic cleanup), assertions must filter by unique ID.

## Shared Test Utilities

Avoid duplicating setup code across integration tests. Use a shared module:

```rust
// tests/common/mod.rs
use sqlx::PgPool;

pub async fn setup_test_db() -> PgPool {
    let database_url = std::env::var("TEST_DATABASE_URL")
        .unwrap_or_else(|_| "postgres://localhost/myapp_test".to_string());
    let pool = PgPool::connect(&database_url).await.unwrap();
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();
    pool
}

pub fn test_user(name: &str, email: &str) -> CreateUserRequest {
    CreateUserRequest {
        name: name.to_string(),
        email: email.to_string(),
    }
}
```

```rust
// tests/test_users.rs
mod common;

#[tokio::test]
async fn create_user_persists_to_database() {
    let pool = common::setup_test_db().await;
    // ...
}
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
