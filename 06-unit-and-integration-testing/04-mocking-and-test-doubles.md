# Mocking and Test Doubles

## Why Test Doubles Exist

When your code depends on external systems — a database, an HTTP API, a message queue, a clock — you face a choice in unit tests. You can use the real dependency (slow, non-deterministic, requires infrastructure) or you can replace it with something simpler that behaves predictably. That replacement is called a *test double*.

Test doubles let you test your logic in isolation, without waiting for network responses, provisioning databases, or worrying about the state of external systems. They keep unit tests fast, deterministic, and focused.

## The Four Types of Test Doubles

### Stubs

A stub returns pre-programmed responses. It does not verify how it was called — it just provides canned answers so the code under test can proceed.

Use a stub when your test needs a dependency to return a specific value but does not care how many times or in what order it was called.

```text
STRUCTURE StubPriceService

// Implement PriceService for StubPriceService
    FUNCTION GET_PRICE(product_id: string) -> float or PriceError
        RETURN Ok(29.99)  // Always returns the same price

TEST order_total_uses_price_from_service
    service <- StubPriceService
    total <- CALCULATE_ORDER_TOTAL(service, "widget", 3)
    ASSERT |total - 89.97| < 0.01
```

### Mocks

A mock verifies that specific interactions occurred. It records calls and lets you assert that certain methods were called with certain arguments a certain number of times.

Use a mock when the *side effect* is the behavior you are testing — for example, verifying that a notification was sent.

```text
STRUCTURE MockNotifier
    calls: list of (string, string)

    FUNCTION NEW() -> MockNotifier
        RETURN MockNotifier { calls: empty list }

    FUNCTION ASSERT_CALLED_ONCE_WITH(user_id: string, message: string)
        ASSERT calls.length = 1, "Expected exactly one call, got " + calls.length
        ASSERT calls[0].first = user_id
        ASSERT calls[0].second = message

// Implement Notifier for MockNotifier
    FUNCTION NOTIFY(user_id: string, message: string)
        APPEND (user_id, message) TO self.calls
```

### Fakes

A fake is a working but simplified implementation. It has real behavior, but takes shortcuts that make it unsuitable for production. The classic example is an in-memory database: it stores and retrieves data correctly, but does not persist anything to disk.

Fakes are more capable than stubs — they maintain state and can handle a variety of inputs without pre-programming each response.

```text
STRUCTURE InMemoryUserRepository
    users: map of (id -> User)
    next_id: counter

// Implement UserRepository for InMemoryUserRepository
    FUNCTION CREATE(name: string, email: string) -> User or RepoError
        id <- self.next_id
        self.next_id <- id + 1
        user <- User { id, name, email }
        self.users[id] <- user
        RETURN Ok(user)

    FUNCTION FIND_BY_ID(id: unsigned integer) -> optional User or RepoError
        RETURN Ok(self.users.GET(id))

    FUNCTION FIND_BY_EMAIL(email: string) -> optional User or RepoError
        RETURN Ok(FIND user IN self.users.values WHERE user.email = email)
```

### Spies

A spy wraps a real or fake implementation and records every call made to it. You use the spy when you want the real behavior *and* want to verify what was called.

```text
STRUCTURE SpyHttpClient
    inner: RealHttpClient
    requests: list of (method: string, url: string)

// Implement HttpClient for SpyHttpClient
    FUNCTION REQUEST(method: string, url: string) -> Response or HttpError
        APPEND (method, url) TO self.requests
        RETURN self.inner.REQUEST(method, url)
```

## Trait-Based Dependency Injection in Rust

Rust does not need a dependency injection framework. The language's trait system provides everything required. The pattern is straightforward:

1. Define the dependency as a trait.
2. Accept the trait (via generics or `dyn Trait`) in your function or struct.
3. Provide a real implementation for production and a test double for tests.

```text
// Step 1: Define the dependency as an interface
INTERFACE Clock
    FUNCTION NOW() -> DateTime

// Step 2: Production implementation
STRUCTURE SystemClock
// Implement Clock for SystemClock
    FUNCTION NOW() -> DateTime
        RETURN current UTC time

// Step 3: Accept the interface in your business logic
FUNCTION IS_MARKET_OPEN(clock: Clock) -> boolean
    now <- clock.NOW()
    hour <- now.hour
    // NYSE hours: 9:30 AM - 4:00 PM ET (simplified)
    RETURN hour ≥ 9 AND hour < 16

// Step 4: Test with a controllable double
STRUCTURE FixedClock
    time: DateTime

// Implement Clock for FixedClock
    FUNCTION NOW() -> DateTime
        RETURN self.time

TEST market_is_open_during_trading_hours
    clock <- FixedClock { time: 2025-06-10 14:30:00 UTC }
    ASSERT IS_MARKET_OPEN(clock) = true

TEST market_is_closed_at_night
    clock <- FixedClock { time: 2025-06-10 22:00:00 UTC }
    ASSERT IS_MARKET_OPEN(clock) = false
```

This pattern works for any external dependency: HTTP clients, email senders, file systems, random number generators, payment gateways. No framework, no macros, no runtime overhead.

### Generics vs. Dynamic Dispatch

You can accept traits via generics (`fn process<T: Clock>(clock: &T)`) or dynamic dispatch (`fn process(clock: &dyn Clock)`). For tests, the choice rarely matters. Use generics when performance is critical (zero-cost abstraction); use `dyn Trait` when you need to store heterogeneous implementations or keep binary size small.

## The `mockall` Crate

For complex traits with many methods, hand-writing test doubles becomes tedious. The `mockall` crate generates mock implementations automatically.

```toml
[dev-dependencies]
mockall = "0.13"
```

### Basic Usage

```text
// Auto-generate mock implementation
INTERFACE PaymentGateway
    FUNCTION CHARGE(amount: float, currency: string) -> string or PaymentError
    FUNCTION REFUND(transaction_id: string) -> void or PaymentError

TEST successful_payment_returns_transaction_id
    mock <- NEW MockPaymentGateway()
    mock.EXPECT_CHARGE()
        .WITH(99.99, "USD")
        .TIMES(1)
        .RETURNING(Ok("txn_abc123"))

    result <- PROCESS_ORDER(mock, 99.99, "USD")
    ASSERT result.transaction_id = "txn_abc123"

TEST failed_payment_propagates_error
    mock <- NEW MockPaymentGateway()
    mock.EXPECT_CHARGE()
        .RETURNING(Err(PaymentError::Declined))

    result <- PROCESS_ORDER(mock, 99.99, "USD")
    ASSERT result IS Err(OrderError::PaymentFailed(_))
```

### Sequence Expectations

Verify that methods are called in a specific order:

```text
TEST checkout_validates_then_charges_then_confirms
    seq <- NEW Sequence()
    mock <- NEW MockCheckoutService()

    mock.EXPECT_VALIDATE()
        .TIMES(1)
        .IN_SEQUENCE(seq)
        .RETURNING(Ok)

    mock.EXPECT_CHARGE()
        .TIMES(1)
        .IN_SEQUENCE(seq)
        .RETURNING(Ok("txn_123"))

    mock.EXPECT_SEND_CONFIRMATION()
        .TIMES(1)
        .IN_SEQUENCE(seq)
        .RETURNING(Ok)

    CHECKOUT(mock, order)
```

## When to Mock vs. Use Real Implementations

This is where most teams go wrong. Over-mocking creates tests that verify your mocks, not your code. Under-mocking creates tests that are slow and flaky.

### Mock These (External Boundaries)

- **Network calls** — HTTP APIs, gRPC services, SMTP servers
- **Databases** — when testing business logic, not query correctness
- **Clocks and timers** — `SystemTime::now()`, `sleep()`
- **Random number generators** — use seeded generators or stubs
- **Payment processors** — never charge real money in tests
- **Third-party SDKs** — AWS, Stripe, Twilio

### Do NOT Mock These (Your Own Code)

- **Your own structs and functions** — if you mock your own `UserService` to test your `OrderService`, you are not testing the real interaction between them. Use integration tests instead.
- **Data transformations** — if a function converts a `Vec<Order>` into a `Report`, just call it with real data.
- **Pure functions** — functions with no side effects never need mocking.

### The Litmus Test

Ask: "Is this dependency something I *own* or something *external*?" Mock external boundaries. Use real implementations for internal code. If your test has more mock setup than assertions, you are probably over-mocking.

## Real-World Example: Testing a Notification Service

Consider a service that sends notifications when an order ships. It depends on an email sender, an SMS sender, and a user preferences store.

```text
INTERFACE EmailSender
    FUNCTION SEND(to: string, subject: string, body: string) -> void or SendError

INTERFACE SmsSender
    FUNCTION SEND(phone: string, message: string) -> void or SendError

INTERFACE UserPreferences
    FUNCTION GET_PREFERENCE(user_id: unsigned integer) -> NotificationPref or PrefError

FUNCTION NOTIFY_SHIPMENT(user_id, order_id, prefs, email, sms) -> void or NotifyError
    pref <- prefs.GET_PREFERENCE(user_id)?
    MATCH pref
        CASE NotificationPref::Email(addr):
            email.SEND(addr, "Your order shipped!", "Order " + order_id + " is on its way.")?
        CASE NotificationPref::Sms(phone):
            sms.SEND(phone, "Order " + order_id + " shipped!")?
        CASE NotificationPref::Both(addr, phone):
            email.SEND(addr, "Your order shipped!", "Order " + order_id + " is on its way.")?
            sms.SEND(phone, "Order " + order_id + " shipped!")?
    RETURN Ok
```

Testing this with hand-written fakes:

```text
STRUCTURE FakePrefs(pref: NotificationPref)

// Implement UserPreferences for FakePrefs
    FUNCTION GET_PREFERENCE(user_id) -> NotificationPref or PrefError
        RETURN Ok(self.pref)

STRUCTURE RecordingEmailSender
    sent: list of (to: string, subject: string, body: string)

// Implement EmailSender for RecordingEmailSender
    FUNCTION SEND(to, subject, body) -> void or SendError
        APPEND (to, subject, body) TO self.sent
        RETURN Ok

TEST email_preference_sends_email_only
    prefs <- FakePrefs(NotificationPref::Email("alice@example.com"))
    email <- RecordingEmailSender { sent: empty list }
    sms <- RecordingSmsSender { sent: empty list }

    NOTIFY_SHIPMENT(1, "ORD-42", prefs, email, sms)

    ASSERT email.sent.length = 1
    ASSERT email.sent[0].to = "alice@example.com"
    ASSERT sms.sent.length = 0  // No SMS sent
```

Each test double here is simple — a few lines of code, no framework. For a trait with two methods, hand-writing a fake is faster than learning a mocking framework's API.

## Common Pitfalls

### Over-Mocking

When every dependency is mocked, the test verifies that your code calls mocks in the right order with the right arguments. But it does not verify that the real dependencies behave the way your mocks pretend. Over-mocked tests pass even when the real system is broken.

### Mock Behavior Drift

If the real `PaymentGateway` starts returning a different response format, your mock will not know. Your tests will pass, but production will break. Mitigate this with contract tests: a shared test suite that runs against both the mock and the real implementation.

### Fragile Mock Expectations

Tests that assert exact call counts, exact argument values, and exact call ordering are brittle. They break when you change implementation details that do not affect the outcome. Only assert what matters for the behavior you are testing.

## Key Takeaways

1. Use traits for dependency injection in Rust. No framework is needed — the type system does the work.
2. Choose the right type of double: stubs for canned responses, mocks for interaction verification, fakes for simplified real behavior, spies for recording calls.
3. Mock external boundaries (APIs, databases, clocks). Do not mock your own code.
4. Hand-written test doubles are often simpler and clearer than framework-generated mocks. Use `mockall` when traits have many methods or complex signatures.
5. If your test has more mock setup than assertions, you are likely over-mocking. Step back and ask whether an integration test would be more appropriate.
6. Guard against mock behavior drift with contract tests that verify your mocks behave like the real implementation.
