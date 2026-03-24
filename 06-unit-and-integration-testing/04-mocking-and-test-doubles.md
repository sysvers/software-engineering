# Mocking and Test Doubles

## Why Test Doubles Exist

When your code depends on external systems — a database, an HTTP API, a message queue, a clock — you face a choice in unit tests. You can use the real dependency (slow, non-deterministic, requires infrastructure) or you can replace it with something simpler that behaves predictably. That replacement is called a *test double*.

Test doubles let you test your logic in isolation, without waiting for network responses, provisioning databases, or worrying about the state of external systems. They keep unit tests fast, deterministic, and focused.

## The Four Types of Test Doubles

### Stubs

A stub returns pre-programmed responses. It does not verify how it was called — it just provides canned answers so the code under test can proceed.

Use a stub when your test needs a dependency to return a specific value but does not care how many times or in what order it was called.

```rust
struct StubPriceService;

impl PriceService for StubPriceService {
    fn get_price(&self, _product_id: &str) -> Result<f64, PriceError> {
        Ok(29.99) // Always returns the same price
    }
}

#[test]
fn order_total_uses_price_from_service() {
    let service = StubPriceService;
    let total = calculate_order_total(&service, "widget", 3);
    assert!((total - 89.97).abs() < 0.01);
}
```

### Mocks

A mock verifies that specific interactions occurred. It records calls and lets you assert that certain methods were called with certain arguments a certain number of times.

Use a mock when the *side effect* is the behavior you are testing — for example, verifying that a notification was sent.

```rust
struct MockNotifier {
    calls: RefCell<Vec<(String, String)>>,
}

impl MockNotifier {
    fn new() -> Self {
        MockNotifier { calls: RefCell::new(vec![]) }
    }

    fn assert_called_once_with(&self, user_id: &str, message: &str) {
        let calls = self.calls.borrow();
        assert_eq!(calls.len(), 1, "Expected exactly one call, got {}", calls.len());
        assert_eq!(calls[0].0, user_id);
        assert_eq!(calls[0].1, message);
    }
}

impl Notifier for MockNotifier {
    fn notify(&self, user_id: &str, message: &str) {
        self.calls.borrow_mut().push((user_id.to_string(), message.to_string()));
    }
}
```

### Fakes

A fake is a working but simplified implementation. It has real behavior, but takes shortcuts that make it unsuitable for production. The classic example is an in-memory database: it stores and retrieves data correctly, but does not persist anything to disk.

Fakes are more capable than stubs — they maintain state and can handle a variety of inputs without pre-programming each response.

```rust
struct InMemoryUserRepository {
    users: RefCell<HashMap<u64, User>>,
    next_id: Cell<u64>,
}

impl UserRepository for InMemoryUserRepository {
    fn create(&self, name: &str, email: &str) -> Result<User, RepoError> {
        let id = self.next_id.get();
        self.next_id.set(id + 1);
        let user = User { id, name: name.to_string(), email: email.to_string() };
        self.users.borrow_mut().insert(id, user.clone());
        Ok(user)
    }

    fn find_by_id(&self, id: u64) -> Result<Option<User>, RepoError> {
        Ok(self.users.borrow().get(&id).cloned())
    }

    fn find_by_email(&self, email: &str) -> Result<Option<User>, RepoError> {
        Ok(self.users.borrow().values().find(|u| u.email == email).cloned())
    }
}
```

### Spies

A spy wraps a real or fake implementation and records every call made to it. You use the spy when you want the real behavior *and* want to verify what was called.

```rust
struct SpyHttpClient {
    inner: RealHttpClient,
    requests: RefCell<Vec<(String, String)>>, // (method, url)
}

impl HttpClient for SpyHttpClient {
    fn request(&self, method: &str, url: &str) -> Result<Response, HttpError> {
        self.requests.borrow_mut().push((method.to_string(), url.to_string()));
        self.inner.request(method, url)
    }
}
```

## Trait-Based Dependency Injection in Rust

Rust does not need a dependency injection framework. The language's trait system provides everything required. The pattern is straightforward:

1. Define the dependency as a trait.
2. Accept the trait (via generics or `dyn Trait`) in your function or struct.
3. Provide a real implementation for production and a test double for tests.

```rust
// Step 1: Define the dependency as a trait
trait Clock {
    fn now(&self) -> DateTime<Utc>;
}

// Step 2: Production implementation
struct SystemClock;

impl Clock for SystemClock {
    fn now(&self) -> DateTime<Utc> {
        Utc::now()
    }
}

// Step 3: Accept the trait in your business logic
fn is_market_open(clock: &dyn Clock) -> bool {
    let now = clock.now();
    let hour = now.hour();
    // NYSE hours: 9:30 AM - 4:00 PM ET (simplified)
    hour >= 9 && hour < 16
}

// Step 4: Test with a controllable double
struct FixedClock {
    time: DateTime<Utc>,
}

impl Clock for FixedClock {
    fn now(&self) -> DateTime<Utc> {
        self.time
    }
}

#[test]
fn market_is_open_during_trading_hours() {
    let clock = FixedClock {
        time: Utc.with_ymd_and_hms(2025, 6, 10, 14, 30, 0).unwrap(),
    };
    assert!(is_market_open(&clock));
}

#[test]
fn market_is_closed_at_night() {
    let clock = FixedClock {
        time: Utc.with_ymd_and_hms(2025, 6, 10, 22, 0, 0).unwrap(),
    };
    assert!(!is_market_open(&clock));
}
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

```rust
use mockall::automock;

#[automock]
trait PaymentGateway {
    fn charge(&self, amount: f64, currency: &str) -> Result<String, PaymentError>;
    fn refund(&self, transaction_id: &str) -> Result<(), PaymentError>;
}

#[test]
fn successful_payment_returns_transaction_id() {
    let mut mock = MockPaymentGateway::new();
    mock.expect_charge()
        .with(eq(99.99), eq("USD"))
        .times(1)
        .returning(|_, _| Ok("txn_abc123".to_string()));

    let result = process_order(&mock, 99.99, "USD");
    assert_eq!(result.unwrap().transaction_id, "txn_abc123");
}

#[test]
fn failed_payment_propagates_error() {
    let mut mock = MockPaymentGateway::new();
    mock.expect_charge()
        .returning(|_, _| Err(PaymentError::Declined));

    let result = process_order(&mock, 99.99, "USD");
    assert!(matches!(result, Err(OrderError::PaymentFailed(_))));
}
```

### Sequence Expectations

Verify that methods are called in a specific order:

```rust
use mockall::Sequence;

#[test]
fn checkout_validates_then_charges_then_confirms() {
    let mut seq = Sequence::new();
    let mut mock = MockCheckoutService::new();

    mock.expect_validate()
        .times(1)
        .in_sequence(&mut seq)
        .returning(|_| Ok(()));

    mock.expect_charge()
        .times(1)
        .in_sequence(&mut seq)
        .returning(|_| Ok("txn_123".into()));

    mock.expect_send_confirmation()
        .times(1)
        .in_sequence(&mut seq)
        .returning(|_| Ok(()));

    checkout(&mock, &order).unwrap();
}
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

```rust
trait EmailSender {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), SendError>;
}

trait SmsSender {
    fn send(&self, phone: &str, message: &str) -> Result<(), SendError>;
}

trait UserPreferences {
    fn get_preference(&self, user_id: u64) -> Result<NotificationPref, PrefError>;
}

fn notify_shipment(
    user_id: u64,
    order_id: &str,
    prefs: &dyn UserPreferences,
    email: &dyn EmailSender,
    sms: &dyn SmsSender,
) -> Result<(), NotifyError> {
    let pref = prefs.get_preference(user_id)?;
    match pref {
        NotificationPref::Email(addr) => {
            email.send(&addr, "Your order shipped!", &format!("Order {} is on its way.", order_id))?;
        }
        NotificationPref::Sms(phone) => {
            sms.send(&phone, &format!("Order {} shipped!", order_id))?;
        }
        NotificationPref::Both(addr, phone) => {
            email.send(&addr, "Your order shipped!", &format!("Order {} is on its way.", order_id))?;
            sms.send(&phone, &format!("Order {} shipped!", order_id))?;
        }
    }
    Ok(())
}
```

Testing this with hand-written fakes:

```rust
struct FakePrefs(NotificationPref);

impl UserPreferences for FakePrefs {
    fn get_preference(&self, _user_id: u64) -> Result<NotificationPref, PrefError> {
        Ok(self.0.clone())
    }
}

struct RecordingEmailSender {
    sent: RefCell<Vec<(String, String, String)>>,
}

impl EmailSender for RecordingEmailSender {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), SendError> {
        self.sent.borrow_mut().push((to.into(), subject.into(), body.into()));
        Ok(())
    }
}

#[test]
fn email_preference_sends_email_only() {
    let prefs = FakePrefs(NotificationPref::Email("alice@example.com".into()));
    let email = RecordingEmailSender { sent: RefCell::new(vec![]) };
    let sms = RecordingSmsSender { sent: RefCell::new(vec![]) };

    notify_shipment(1, "ORD-42", &prefs, &email, &sms).unwrap();

    assert_eq!(email.sent.borrow().len(), 1);
    assert_eq!(email.sent.borrow()[0].0, "alice@example.com");
    assert_eq!(sms.sent.borrow().len(), 0); // No SMS sent
}
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
