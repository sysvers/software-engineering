# Interface Design

Interfaces (traits in Rust) define contracts between components. Good interfaces enable flexibility, testability, and independent evolution. Bad interfaces create coupling that makes every change ripple across the codebase.

## Program to Interfaces, Not Implementations

The single most impactful interface design principle: depend on abstractions, not concrete types.

```rust
// Bad — tightly coupled to PostgreSQL
fn create_order(pool: &PgPool, order: Order) -> Result<OrderId, sqlx::Error> {
    // SQL query here
}

// Good — depends on an abstraction
trait OrderRepository {
    fn save(&self, order: &Order) -> Result<OrderId, RepositoryError>;
    fn find_by_id(&self, id: OrderId) -> Result<Option<Order>, RepositoryError>;
    fn find_by_user(&self, user_id: UserId) -> Result<Vec<Order>, RepositoryError>;
}

// Now you can swap implementations without changing callers
struct PostgresOrderRepo { pool: PgPool }
struct InMemoryOrderRepo { orders: HashMap<OrderId, Order> }  // For testing
```

**Why this matters:** The `OrderService` that calls `OrderRepository` does not know or care whether orders live in PostgreSQL, DynamoDB, or an in-memory HashMap. You can test the service with the in-memory implementation (fast, no database setup) and run the real database in production.

### Implementing Traits for Swappable Backends

```rust
impl OrderRepository for PostgresOrderRepo {
    fn save(&self, order: &Order) -> Result<OrderId, RepositoryError> {
        sqlx::query!("INSERT INTO orders ...")
            .execute(&self.pool)
            .await
            .map_err(|e| RepositoryError::Storage(e.to_string()))?;
        Ok(order.id)
    }

    fn find_by_id(&self, id: OrderId) -> Result<Option<Order>, RepositoryError> {
        sqlx::query_as!(Order, "SELECT * FROM orders WHERE id = $1", id.0)
            .fetch_optional(&self.pool)
            .await
            .map_err(|e| RepositoryError::Storage(e.to_string()))
    }

    fn find_by_user(&self, user_id: UserId) -> Result<Vec<Order>, RepositoryError> {
        sqlx::query_as!(Order, "SELECT * FROM orders WHERE user_id = $1", user_id.0)
            .fetch_all(&self.pool)
            .await
            .map_err(|e| RepositoryError::Storage(e.to_string()))
    }
}

impl OrderRepository for InMemoryOrderRepo {
    fn save(&self, order: &Order) -> Result<OrderId, RepositoryError> {
        self.orders.insert(order.id, order.clone());
        Ok(order.id)
    }

    fn find_by_id(&self, id: OrderId) -> Result<Option<Order>, RepositoryError> {
        Ok(self.orders.get(&id).cloned())
    }

    fn find_by_user(&self, user_id: UserId) -> Result<Vec<Order>, RepositoryError> {
        Ok(self.orders.values()
            .filter(|o| o.user_id == user_id)
            .cloned()
            .collect())
    }
}
```

## Interface Segregation

Keep interfaces small and focused. A consumer should not be forced to depend on methods it does not use.

```rust
// Bad — one massive trait
trait UserService {
    fn create(&self, user: NewUser) -> Result<User, Error>;
    fn find(&self, id: UserId) -> Result<Option<User>, Error>;
    fn update(&self, id: UserId, changes: UserUpdate) -> Result<User, Error>;
    fn delete(&self, id: UserId) -> Result<(), Error>;
    fn authenticate(&self, email: &str, password: &str) -> Result<Token, Error>;
    fn reset_password(&self, email: &str) -> Result<(), Error>;
    fn send_verification_email(&self, user: &User) -> Result<(), Error>;
    fn upload_avatar(&self, user: &User, image: &[u8]) -> Result<Url, Error>;
}

// Good — split by concern
trait UserRepository {
    fn save(&self, user: &User) -> Result<(), Error>;
    fn find_by_id(&self, id: UserId) -> Result<Option<User>, Error>;
}

trait AuthService {
    fn authenticate(&self, email: &str, password: &str) -> Result<Token, Error>;
    fn reset_password(&self, email: &str) -> Result<(), Error>;
}

trait AvatarService {
    fn upload(&self, user_id: UserId, image: &[u8]) -> Result<Url, Error>;
}
```

**Why split?** The handler that displays a user profile only needs `UserRepository`. The login handler only needs `AuthService`. Neither should depend on avatar upload logic. When you change avatar storage from S3 to GCS, only `AvatarService` implementors change — not every consumer of the old monolithic `UserService`.

## Make Impossible States Unrepresentable

Rust's enum system is uniquely powerful for encoding valid states into the type system. If a state combination is invalid, the compiler should reject it.

### The Problem with Optional Fields

```rust
// Bad — caller must remember to check status before accessing fields
struct Payment {
    status: PaymentStatus,
    transaction_id: Option<String>,  // Only present if Completed
    error_message: Option<String>,   // Only present if Failed
    refund_id: Option<String>,       // Only present if Refunded
}
```

This struct allows nonsensical states: a `Pending` payment with a `transaction_id`, or a `Completed` payment with an `error_message`. Every consumer must check combinations manually and hope they get it right.

### The Solution: Enums With Data

```rust
// Good — the type system enforces valid combinations
enum Payment {
    Pending { created_at: DateTime<Utc> },
    Completed { transaction_id: String, completed_at: DateTime<Utc> },
    Failed { error_message: String, failed_at: DateTime<Utc> },
    Refunded { refund_id: String, original_transaction_id: String },
}
```

Now it is structurally impossible to have a `Pending` payment with a `transaction_id`. The compiler enforces it.

### Newtype Pattern for Type Safety

Prevent mixing up values that have the same underlying type:

```rust
// Bad — stringly typed, easy to mix up arguments
fn transfer(from: String, to: String, amount: f64) {}
// transfer(to_account, from_account, amount) — compiles, silently wrong

// Good — newtypes prevent mixing
struct AccountId(String);
struct Money(Decimal);

fn transfer(from: AccountId, to: AccountId, amount: Money) {}
// transfer(to_account, from_account, amount) — compile error if types differ
```

### Builder Pattern for Complex Construction

When an object requires many parameters, some optional, use a builder to make construction clear and safe:

```rust
struct QueryBuilder {
    table: String,
    conditions: Vec<Condition>,
    limit: Option<usize>,
    offset: Option<usize>,
    order_by: Option<(String, Direction)>,
}

impl QueryBuilder {
    fn new(table: &str) -> Self {
        Self { table: table.into(), conditions: vec![], limit: None, offset: None, order_by: None }
    }

    fn where_eq(mut self, field: &str, value: Value) -> Self {
        self.conditions.push(Condition::Eq(field.into(), value));
        self
    }

    fn limit(mut self, n: usize) -> Self { self.limit = Some(n); self }
    fn offset(mut self, n: usize) -> Self { self.offset = Some(n); self }

    fn build(self) -> Query { /* ... */ }
}

// Usage is readable and hard to get wrong
let query = QueryBuilder::new("orders")
    .where_eq("user_id", user_id.into())
    .where_eq("status", "active".into())
    .limit(10)
    .build();
```

## Structured Error Types

Use enums for errors so callers can handle specific failure modes:

```rust
// Bad
fn process() -> Result<(), String> { /* ... */ }

// Good
enum OrderError {
    NotFound(OrderId),
    InvalidState { current: &'static str, attempted_action: &'static str },
    PaymentFailed(PaymentError),
    InsufficientInventory { product_id: ProductId, requested: u32, available: u32 },
}

impl std::fmt::Display for OrderError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::NotFound(id) => write!(f, "Order {} not found", id.0),
            Self::InvalidState { current, attempted_action } =>
                write!(f, "Cannot {} order in {} state", attempted_action, current),
            Self::PaymentFailed(e) => write!(f, "Payment failed: {}", e),
            Self::InsufficientInventory { product_id, requested, available } =>
                write!(f, "Product {}: requested {}, only {} available",
                    product_id.0, requested, available),
        }
    }
}
```

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| Trait-heavy design | Flexible, testable, swappable | More indirection, harder to follow |
| Concrete types only | Direct, easy to understand | Hard to test, hard to swap |
| Type-driven design (newtype, typestate) | Compile-time safety, prevents bug classes | More boilerplate, learning curve |
| Stringly-typed | Quick to write | Bugs at runtime, no compiler help |
