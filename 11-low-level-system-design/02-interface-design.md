# Interface Design

Interfaces (traits in Rust) define contracts between components. Good interfaces enable flexibility, testability, and independent evolution. Bad interfaces create coupling that makes every change ripple across the codebase.

## Program to Interfaces, Not Implementations

The single most impactful interface design principle: depend on abstractions, not concrete types.

```text
// Bad -- tightly coupled to PostgreSQL
FUNCTION CREATE_ORDER(pool: PgPool, order: Order) → Result<OrderId, SqlError>
    // SQL query here

// Good -- depends on an abstraction
INTERFACE OrderRepository
    FUNCTION SAVE(order: Order) → Result<OrderId, RepositoryError>
    FUNCTION FIND_BY_ID(id: OrderId) → Result<Optional<Order>, RepositoryError>
    FUNCTION FIND_BY_USER(user_id: UserId) → Result<List<Order>, RepositoryError>

// Now you can swap implementations without changing callers
RECORD PostgresOrderRepo { pool: PgPool }
RECORD InMemoryOrderRepo { orders: Map<OrderId, Order> }  // For testing
```

**Why this matters:** The `OrderService` that calls `OrderRepository` does not know or care whether orders live in PostgreSQL, DynamoDB, or an in-memory HashMap. You can test the service with the in-memory implementation (fast, no database setup) and run the real database in production.

### Implementing Traits for Swappable Backends

```text
IMPLEMENTS OrderRepository FOR PostgresOrderRepo
    FUNCTION SAVE(order: Order) → Result<OrderId, RepositoryError>
        EXECUTE SQL "INSERT INTO orders ..." USING self.pool
        IF error THEN RETURN Error(StorageError(error.message))
        RETURN order.id

    FUNCTION FIND_BY_ID(id: OrderId) → Result<Optional<Order>, RepositoryError>
        result ← EXECUTE SQL "SELECT * FROM orders WHERE id = ?" WITH id USING self.pool
        IF error THEN RETURN Error(StorageError(error.message))
        RETURN result

    FUNCTION FIND_BY_USER(user_id: UserId) → Result<List<Order>, RepositoryError>
        results ← EXECUTE SQL "SELECT * FROM orders WHERE user_id = ?" WITH user_id USING self.pool
        IF error THEN RETURN Error(StorageError(error.message))
        RETURN results

IMPLEMENTS OrderRepository FOR InMemoryOrderRepo
    FUNCTION SAVE(order: Order) → Result<OrderId, RepositoryError>
        self.orders[order.id] ← COPY(order)
        RETURN order.id

    FUNCTION FIND_BY_ID(id: OrderId) → Result<Optional<Order>, RepositoryError>
        RETURN COPY(self.orders.GET(id))

    FUNCTION FIND_BY_USER(user_id: UserId) → Result<List<Order>, RepositoryError>
        RETURN [COPY(o) FOR o IN self.orders.VALUES() WHERE o.user_id = user_id]
```

## Interface Segregation

Keep interfaces small and focused. A consumer should not be forced to depend on methods it does not use.

```text
// Bad -- one massive interface
INTERFACE UserService
    FUNCTION CREATE(user: NewUser) → Result<User, Error>
    FUNCTION FIND(id: UserId) → Result<Optional<User>, Error>
    FUNCTION UPDATE(id: UserId, changes: UserUpdate) → Result<User, Error>
    FUNCTION DELETE(id: UserId) → Result<Void, Error>
    FUNCTION AUTHENTICATE(email: String, password: String) → Result<Token, Error>
    FUNCTION RESET_PASSWORD(email: String) → Result<Void, Error>
    FUNCTION SEND_VERIFICATION_EMAIL(user: User) → Result<Void, Error>
    FUNCTION UPLOAD_AVATAR(user: User, image: Bytes) → Result<Url, Error>

// Good -- split by concern
INTERFACE UserRepository
    FUNCTION SAVE(user: User) → Result<Void, Error>
    FUNCTION FIND_BY_ID(id: UserId) → Result<Optional<User>, Error>

INTERFACE AuthService
    FUNCTION AUTHENTICATE(email: String, password: String) → Result<Token, Error>
    FUNCTION RESET_PASSWORD(email: String) → Result<Void, Error>

INTERFACE AvatarService
    FUNCTION UPLOAD(user_id: UserId, image: Bytes) → Result<Url, Error>
```

**Why split?** The handler that displays a user profile only needs `UserRepository`. The login handler only needs `AuthService`. Neither should depend on avatar upload logic. When you change avatar storage from S3 to GCS, only `AvatarService` implementors change — not every consumer of the old monolithic `UserService`.

## Make Impossible States Unrepresentable

Rust's enum system is uniquely powerful for encoding valid states into the type system. If a state combination is invalid, the compiler should reject it.

### The Problem with Optional Fields

```text
// Bad -- caller must remember to check status before accessing fields
RECORD Payment
    status: PaymentStatus
    transaction_id: Optional<String>  // Only present if Completed
    error_message: Optional<String>   // Only present if Failed
    refund_id: Optional<String>       // Only present if Refunded
```

This struct allows nonsensical states: a `Pending` payment with a `transaction_id`, or a `Completed` payment with an `error_message`. Every consumer must check combinations manually and hope they get it right.

### The Solution: Enums With Data

```text
// Good -- the type system enforces valid combinations
ENUM Payment
    Pending { created_at: DateTime }
    Completed { transaction_id: String, completed_at: DateTime }
    Failed { error_message: String, failed_at: DateTime }
    Refunded { refund_id: String, original_transaction_id: String }
```

Now it is structurally impossible to have a `Pending` payment with a `transaction_id`. The compiler enforces it.

### Newtype Pattern for Type Safety

Prevent mixing up values that have the same underlying type:

```text
// Bad -- stringly typed, easy to mix up arguments
FUNCTION TRANSFER(from: String, to: String, amount: Float)
// TRANSFER(to_account, from_account, amount) -- compiles, silently wrong

// Good -- newtypes prevent mixing
TYPE AccountId = WRAPPER(String)
TYPE Money = WRAPPER(Decimal)

FUNCTION TRANSFER(from: AccountId, to: AccountId, amount: Money)
// TRANSFER(to_account, from_account, amount) -- type error if types differ
```

### Builder Pattern for Complex Construction

When an object requires many parameters, some optional, use a builder to make construction clear and safe:

```text
RECORD QueryBuilder
    table: String
    conditions: List<Condition>
    limit: Optional<Integer>
    offset: Optional<Integer>
    order_by: Optional<(String, Direction)>

FUNCTION NEW_QUERY_BUILDER(table: String) → QueryBuilder
    RETURN QueryBuilder { table ← table, conditions ← [], limit ← NONE, offset ← NONE, order_by ← NONE }

FUNCTION WHERE_EQ(builder: QueryBuilder, field: String, value: Value) → QueryBuilder
    APPEND Condition.Eq(field, value) TO builder.conditions
    RETURN builder

FUNCTION LIMIT(builder: QueryBuilder, n: Integer) → QueryBuilder
    builder.limit ← n
    RETURN builder

FUNCTION OFFSET(builder: QueryBuilder, n: Integer) → QueryBuilder
    builder.offset ← n
    RETURN builder

FUNCTION BUILD(builder: QueryBuilder) → Query
    // ...

// Usage is readable and hard to get wrong
query ← BUILD(
    LIMIT(
        WHERE_EQ(
            WHERE_EQ(
                NEW_QUERY_BUILDER("orders"),
                "user_id", user_id),
            "status", "active"),
        10))
```

## Structured Error Types

Use enums for errors so callers can handle specific failure modes:

```text
// Bad
FUNCTION PROCESS() → Result<Void, String>

// Good
ENUM OrderError
    NotFound { id: OrderId }
    InvalidState { current: String, attempted_action: String }
    PaymentFailed { error: PaymentError }
    InsufficientInventory { product_id: ProductId, requested: Integer, available: Integer }

FUNCTION TO_STRING(err: OrderError) → String
    MATCH err
        CASE NotFound { id }:
            RETURN "Order " + id + " not found"
        CASE InvalidState { current, attempted_action }:
            RETURN "Cannot " + attempted_action + " order in " + current + " state"
        CASE PaymentFailed { error }:
            RETURN "Payment failed: " + TO_STRING(error)
        CASE InsufficientInventory { product_id, requested, available }:
            RETURN "Product " + product_id + ": requested " + requested + ", only " + available + " available"
```

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| Trait-heavy design | Flexible, testable, swappable | More indirection, harder to follow |
| Concrete types only | Direct, easy to understand | Hard to test, hard to swap |
| Type-driven design (newtype, typestate) | Compile-time safety, prevents bug classes | More boilerplate, learning curve |
| Stringly-typed | Quick to write | Bugs at runtime, no compiler help |
