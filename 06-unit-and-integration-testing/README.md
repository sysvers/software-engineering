# 06 - Unit & Integration Testing

## Diagrams

![TDD Cycle](diagrams/tdd-cycle.png)

![Test Pyramid](diagrams/test-pyramid.png)


## Concepts

### The Test Pyramid

The test pyramid (coined by Mike Cohn) describes the ideal distribution of test types:

```
        /  \        ← Few: E2E tests (slow, expensive, brittle)
       /    \
      /------\      ← Some: Integration tests (moderate speed)
     /        \
    /----------\    ← Many: Unit tests (fast, cheap, focused)
```

- **Unit tests** form the base — fast, isolated, testing individual functions/modules
- **Integration tests** in the middle — verify that components work together
- **End-to-end tests** at the top — simulate real user behavior (covered in Topic 07)

The pyramid shape means: write *many* unit tests, *some* integration tests, and *few* E2E tests. Inverting this pyramid (many E2E, few unit tests) leads to slow CI, flaky tests, and painful debugging.

### Unit Testing

A unit test verifies a single unit of behavior in isolation. In Rust, a "unit" is typically a function or a method.

**Characteristics of a good unit test:**
- **Fast** — Runs in milliseconds. A suite of 1,000 unit tests should finish in seconds.
- **Isolated** — Doesn't depend on databases, network, filesystem, or other tests
- **Deterministic** — Same input, same result. Every time. No randomness, no time-dependence.
- **Focused** — Tests one behavior. When it fails, you know exactly what's broken.

**Example in Rust:**

```rust
// src/pricing.rs
pub fn calculate_discount(price: f64, discount_percent: f64) -> f64 {
    if discount_percent < 0.0 || discount_percent > 100.0 {
        return price;
    }
    price * (1.0 - discount_percent / 100.0)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn applies_percentage_discount() {
        let result = calculate_discount(100.0, 20.0);
        assert!((result - 80.0).abs() < f64::EPSILON);
    }

    #[test]
    fn zero_discount_returns_original_price() {
        let result = calculate_discount(50.0, 0.0);
        assert!((result - 50.0).abs() < f64::EPSILON);
    }

    #[test]
    fn full_discount_returns_zero() {
        let result = calculate_discount(100.0, 100.0);
        assert!((result - 0.0).abs() < f64::EPSILON);
    }

    #[test]
    fn negative_discount_returns_original_price() {
        let result = calculate_discount(100.0, -10.0);
        assert!((result - 100.0).abs() < f64::EPSILON);
    }

    #[test]
    fn discount_over_100_returns_original_price() {
        let result = calculate_discount(100.0, 150.0);
        assert!((result - 100.0).abs() < f64::EPSILON);
    }
}
```

**Test naming:** Test names should describe the behavior, not the implementation. `applies_percentage_discount` is better than `test_calculate_discount_1`. When a test fails, the name should tell you what broke.

### Arrange-Act-Assert (AAA)

Structure every test in three sections:

```rust
#[test]
fn deactivated_user_cannot_place_order() {
    // Arrange — set up the test data
    let user = User { id: 1, active: false };
    let order = Order::new(vec![Item::new("Widget", 9.99)]);

    // Act — perform the action being tested
    let result = place_order(&user, &order);

    // Assert — verify the expected outcome
    assert!(result.is_err());
    assert_eq!(result.unwrap_err(), OrderError::InactiveUser);
}
```

This pattern makes tests readable and consistent. Anyone can look at a test and immediately understand the scenario, the action, and the expected result.

### Test-Driven Development (TDD)

TDD inverts the typical workflow: write the test *first*, then write the code to make it pass.

**The TDD cycle (Red-Green-Refactor):**

1. **Red** — Write a failing test for the behavior you want
2. **Green** — Write the minimum code to make the test pass
3. **Refactor** — Clean up the code while keeping tests green

```
Write failing test → Write code to pass → Refactor → Repeat
      (Red)              (Green)         (Clean)
```

**Example TDD workflow:**

```rust
// Step 1: RED — Write the test first (it won't compile yet)
#[test]
fn empty_cart_has_zero_total() {
    let cart = ShoppingCart::new();
    assert_eq!(cart.total(), 0.0);
}

// Step 2: GREEN — Write minimum code to pass
pub struct ShoppingCart;

impl ShoppingCart {
    pub fn new() -> Self { ShoppingCart }
    pub fn total(&self) -> f64 { 0.0 }
}

// Step 3: RED — Next test
#[test]
fn cart_with_one_item_returns_item_price() {
    let mut cart = ShoppingCart::new();
    cart.add_item("Widget", 9.99);
    assert_eq!(cart.total(), 9.99);
}

// Step 4: GREEN — Extend to pass
pub struct ShoppingCart {
    items: Vec<(String, f64)>,
}

impl ShoppingCart {
    pub fn new() -> Self {
        ShoppingCart { items: vec![] }
    }
    pub fn add_item(&mut self, name: &str, price: f64) {
        self.items.push((name.to_string(), price));
    }
    pub fn total(&self) -> f64 {
        self.items.iter().map(|(_, price)| price).sum()
    }
}
```

**When TDD works well:**
- Well-defined business logic (pricing, validation, calculations)
- When you need to think through edge cases before implementing
- Bug fixes — write a test that reproduces the bug, then fix it

**When TDD is awkward:**
- Exploratory code where you don't yet know the design
- UI code with rapidly changing layouts
- Glue code that mostly calls other services

### Property-Based Testing

Instead of writing specific examples, property-based testing generates hundreds of random inputs and verifies that certain *properties* always hold.

**Example with `proptest` in Rust:**

```rust
use proptest::prelude::*;

// Traditional test: specific cases
#[test]
fn sort_works_for_specific_input() {
    assert_eq!(sort(vec![3, 1, 2]), vec![1, 2, 3]);
}

// Property-based test: any input
proptest! {
    #[test]
    fn sorted_output_has_same_length(input: Vec<i32>) {
        let sorted = sort(input.clone());
        assert_eq!(sorted.len(), input.len());
    }

    #[test]
    fn sorted_output_is_ordered(input: Vec<i32>) {
        let sorted = sort(input);
        for window in sorted.windows(2) {
            assert!(window[0] <= window[1]);
        }
    }

    #[test]
    fn sorted_output_contains_same_elements(input: Vec<i32>) {
        let mut sorted = sort(input.clone());
        let mut original = input;
        original.sort();
        assert_eq!(sorted, original);
    }
}
```

Property-based tests find edge cases you'd never think to write manually: empty inputs, very large inputs, boundary values, unicode strings, negative numbers.

### Mocking, Stubbing & Test Doubles

When code depends on external systems (databases, APIs, clocks), you need test doubles to isolate the unit under test.

**Types of test doubles:**

| Type | Purpose | Example |
|------|---------|---------|
| **Stub** | Returns pre-programmed responses | A payment gateway that always returns "approved" |
| **Mock** | Verifies that specific interactions happened | Assert that `send_email()` was called exactly once |
| **Fake** | A working but simplified implementation | An in-memory database instead of PostgreSQL |
| **Spy** | Records calls for later assertions | Logs all HTTP requests made during the test |

**Example using traits for dependency injection in Rust:**

```rust
// Define the dependency as a trait
trait EmailSender {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), EmailError>;
}

// Production implementation
struct SmtpEmailSender { /* ... */ }
impl EmailSender for SmtpEmailSender {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), EmailError> {
        // Actually sends email via SMTP
        todo!()
    }
}

// Test double — a fake that records calls
struct FakeEmailSender {
    sent_emails: RefCell<Vec<(String, String, String)>>,
}

impl EmailSender for FakeEmailSender {
    fn send(&self, to: &str, subject: &str, body: &str) -> Result<(), EmailError> {
        self.sent_emails.borrow_mut().push((
            to.to_string(),
            subject.to_string(),
            body.to_string(),
        ));
        Ok(())
    }
}

// The function under test accepts any EmailSender
fn register_user(
    email: &str,
    sender: &dyn EmailSender,
) -> Result<User, RegistrationError> {
    let user = User::create(email)?;
    sender.send(email, "Welcome!", "Thanks for registering.")?;
    Ok(user)
}

#[test]
fn registration_sends_welcome_email() {
    let sender = FakeEmailSender { sent_emails: RefCell::new(vec![]) };

    register_user("alice@example.com", &sender).unwrap();

    let emails = sender.sent_emails.borrow();
    assert_eq!(emails.len(), 1);
    assert_eq!(emails[0].0, "alice@example.com");
    assert_eq!(emails[0].1, "Welcome!");
}
```

### Code Coverage

Code coverage measures what percentage of your code is executed by tests. Common metrics:

- **Line coverage** — What % of lines were executed?
- **Branch coverage** — What % of if/else/match branches were taken?
- **Function coverage** — What % of functions were called?

**The coverage trap:** High coverage ≠ well-tested code. You can have 100% line coverage and still miss critical bugs if your assertions are weak.

```rust
// 100% coverage, 0% useful testing
#[test]
fn covers_everything_tests_nothing() {
    let _ = process_payment(100.0, "USD"); // No assertions!
}
```

**Useful coverage targets:**
- 70-80% is a healthy target for most projects
- 90%+ makes sense for critical business logic
- 100% is rarely worth pursuing — the last 10% is usually error handling and edge cases that cost disproportionate effort

### Integration Testing

Integration tests verify that multiple components work together correctly. They test the *seams* between units.

**What integration tests cover:**
- Database queries actually return correct data
- API endpoints correctly parse requests and return responses
- Message queue consumers correctly process messages
- Service A can actually call Service B

**Example in Rust (testing with a real database):**

```rust
#[tokio::test]
async fn user_can_be_created_and_retrieved() {
    // Arrange — set up a test database
    let pool = setup_test_database().await;

    // Act — use the real repository
    let repo = UserRepository::new(pool.clone());
    let created = repo.create_user("alice@example.com", "Alice").await.unwrap();
    let retrieved = repo.get_user(created.id).await.unwrap();

    // Assert — verify the roundtrip
    assert_eq!(retrieved.email, "alice@example.com");
    assert_eq!(retrieved.name, "Alice");

    // Cleanup
    teardown_test_database(pool).await;
}
```

**Unit vs Integration tests — when to use which:**

| Scenario | Unit test | Integration test |
|----------|-----------|-----------------|
| Business logic (calculations, validation) | ✅ | |
| Database queries return correct data | | ✅ |
| API request/response handling | | ✅ |
| Error handling paths | ✅ | |
| Service-to-service communication | | ✅ |
| Algorithm correctness | ✅ | |
| Configuration loading | | ✅ |

## Business Value

- **Regression prevention**: A comprehensive test suite catches bugs introduced by new changes before they reach users. Without tests, every code change is a gamble.
- **Confidence to refactor**: Tests act as a safety net. Teams with good tests refactor aggressively, keeping the codebase healthy. Teams without tests are afraid to touch working code.
- **Faster debugging**: When a test fails, it narrows the problem to a specific unit. Without tests, debugging starts with "something is broken somewhere."
- **Living documentation**: Tests describe what the code is *supposed* to do. When written well, they're the most accurate documentation — unlike comments, tests fail when they're wrong.
- **Deployment confidence**: Teams with strong test suites deploy multiple times per day. Teams without tests deploy weekly (or monthly) because each deployment is risky.
- **Quantifiable impact**: Google found that their most productive teams had test suites that ran in under 5 minutes and caught 85%+ of bugs before they reached code review.

## Real-World Examples

### Google's Testing Culture
Google expects engineers to write tests for every change. Their internal code review tool flags changes without tests. They categorize tests as "small" (unit, <1 second), "medium" (integration, <5 minutes), and "large" (E2E, <15 minutes). The vast majority are small tests. They've found that the return on investment of testing peaks at the unit test level — the cheapest tests to write and maintain.

### Netflix's Test Strategy
Netflix uses a "testing in production" philosophy — but not in the way you might think. They still write unit and integration tests, but they also run extensive canary deployments and chaos experiments in production. Their approach: unit tests catch logic errors, integration tests catch contract violations, and production testing catches environmental issues that no test environment can replicate.

### Airbnb's Testing Pyramid Inversion
Airbnb discovered they had an inverted test pyramid — most of their tests were slow, flaky E2E tests (using Selenium). These tests took hours to run and failed randomly due to timing issues. They invested in rebalancing: deleting flaky E2E tests, replacing them with fast integration and unit tests, and only keeping E2E tests for critical user flows (booking, payment). The result: CI time dropped from hours to minutes, and test reliability improved dramatically.

### How TDD Saved a Financial Trading Platform
A financial services firm adopted TDD for their trading engine. When a regulatory change required recalculating settlement dates differently, they modified the tests first, then updated the code. Every edge case (holidays, weekends, cross-timezone trades) was captured in tests before the code changed. The migration was completed in 2 days with zero production incidents — a process that previously took 2 weeks and always caused bugs.

## Common Mistakes & Pitfalls

- **Testing implementation instead of behavior** — Tests that break when you refactor (but behavior hasn't changed) are testing the wrong thing. Test *what* the code does, not *how* it does it.

- **Flaky tests** — Tests that sometimes pass and sometimes fail. Usually caused by: timing dependencies, shared test state, random data, or network calls. Fix or delete them — flaky tests erode trust in the entire suite.

- **Slow test suites** — A test suite that takes 30 minutes kills developer productivity. If tests are slow, developers stop running them locally and wait for CI, adding hours to the feedback loop.

- **Testing trivial code** — Writing tests for getters, setters, and simple constructors. Focus testing effort on code with business logic, conditional paths, and error handling.

- **Mocking everything** — Over-mocking creates tests that test your mocks, not your code. Mock external dependencies (APIs, databases); don't mock your own code unless necessary.

- **No test maintenance** — Tests need maintenance like production code. When you change a feature, update the tests. Delete tests for removed features.

- **Ignoring test readability** — Tests are documentation. If a test is hard to understand, it won't be maintained. Use clear names, the AAA pattern, and helpful assertion messages.

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **TDD** | Drives good design, high coverage, catches bugs early | Slower initial development, awkward for exploratory work |
| **Test-after** | Faster initial coding, works when design is uncertain | Lower coverage, tests may be superficial, hard to test tightly-coupled code |
| **Property-based testing** | Finds edge cases you wouldn't think of | Harder to write, slower to run, can be hard to debug failures |
| **Heavy mocking** | Fast tests, full isolation | Tests may not reflect reality, maintenance burden |
| **Integration-heavy** | Tests real behavior end-to-end | Slow, requires infrastructure, harder to debug failures |

## When to Use / When Not to Use

**Unit tests — always use for:**
- Business logic (pricing, validation, calculations)
- Pure functions with well-defined inputs/outputs
- Code with multiple conditional paths
- Bug fixes (write a test that reproduces the bug first)

**Unit tests — lower priority for:**
- Simple CRUD with no business logic
- Thin wrappers around external libraries
- UI layout code

**Integration tests — use for:**
- Database queries (verify the SQL works with real data)
- API endpoints (verify request parsing and response formatting)
- Service-to-service communication
- Configuration and startup logic

**TDD — use for:**
- Well-understood domains with clear requirements
- Financial calculations, rule engines, parsers
- When you want the design to emerge from requirements

**TDD — skip for:**
- Exploratory prototyping where the design is unknown
- Quick scripts or one-off tools

## Key Takeaways

1. The test pyramid exists for a reason: many unit tests (fast, cheap), fewer integration tests, fewest E2E tests. Don't invert it.
2. Good tests are fast, isolated, deterministic, and focused. If a test has any of these problems, fix the test.
3. Test behavior, not implementation. Your tests should survive refactoring.
4. Property-based testing finds bugs that example-based tests miss. Use it for logic-heavy code.
5. Use traits for dependency injection in Rust — this enables testing without mocking frameworks.
6. Code coverage is a useful indicator, not a goal. 80% meaningful coverage beats 100% superficial coverage.
7. The best test suite is one that developers actually run. Keep it fast (<5 minutes) and reliable (no flakes).

## Further Reading

- **Books:**
  - *Unit Testing Principles, Practices, and Patterns* — Vladimir Khorikov (2020) — The best modern book on unit testing
  - *Test-Driven Development: By Example* — Kent Beck (2002) — The original TDD book
  - *Growing Object-Oriented Software, Guided by Tests* — Steve Freeman & Nat Pryce (2009) — TDD with mocks and outside-in design

- **Papers & Articles:**
  - [Google Testing Blog](https://testing.googleblog.com/) — Practical testing insights from Google
  - [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) — Ham Vocke's comprehensive guide
  - [Testing in Rust](https://doc.rust-lang.org/book/ch11-00-testing.html) — The Rust Book's testing chapter

- **Tools:**
  - [proptest](https://crates.io/crates/proptest) — Property-based testing for Rust
  - [mockall](https://crates.io/crates/mockall) — Mocking framework for Rust
  - [cargo-tarpaulin](https://crates.io/crates/cargo-tarpaulin) — Code coverage for Rust
  - [nextest](https://nexte.st/) — A faster test runner for Rust
