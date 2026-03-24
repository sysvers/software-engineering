# Unit Testing Fundamentals

## What Is a Unit Test?

A unit test verifies a single unit of behavior in isolation. In Rust, a "unit" is typically a function or a method on a struct. The goal is to confirm that one small piece of logic does exactly what it should, independent of databases, networks, filesystems, or any other external system.

Unit tests form the base of the test pyramid (coined by Mike Cohn). You write *many* of them because they are cheap to create, fast to run, and precise when they fail. Inverting the pyramid — relying heavily on slow end-to-end tests instead of unit tests — leads to painful CI times, flaky suites, and vague failure messages.

```
        /  \        <- Few: E2E tests (slow, expensive, brittle)
       /    \
      /------\      <- Some: Integration tests (moderate speed)
     /        \
    /----------\    <- Many: Unit tests (fast, cheap, focused)
```

## The Four Qualities of a Good Unit Test

### Fast

A single unit test should run in milliseconds. A suite of 1,000 unit tests should complete in seconds, not minutes. Speed matters because developers need to run tests constantly — after every small change, before every commit. If the suite takes 30 minutes, developers stop running it locally and rely on CI, adding hours to the feedback loop.

Practical guideline: if a unit test takes longer than 100ms, it is probably not a unit test. It may be calling a database, making a network request, or doing heavy I/O. Move it to integration tests or remove the external dependency.

### Isolated

A unit test must not depend on anything outside the unit under test. No database connections, no HTTP calls, no filesystem reads, no shared mutable state between tests. If test A must run before test B for test B to pass, both tests are broken.

In Rust, `cargo test` runs tests in parallel by default. This is a natural enforcement mechanism — tests that share state will fail intermittently. Embrace this parallelism rather than fighting it with `--test-threads=1`.

### Deterministic

Same input, same result. Every single time. A test that passes 99 times out of 100 is worse than a test that fails consistently, because intermittent failures erode trust in the entire suite. Sources of non-determinism to avoid:

- Current time (`SystemTime::now()`) — inject a clock dependency instead
- Random number generators — use seeded RNGs in tests
- Network calls — use test doubles (see topic 04)
- File ordering — don't assume directory listing order
- Floating-point comparison — use epsilon-based comparisons

### Focused

A unit test should test one behavior. When it fails, the name and assertion should tell you exactly what broke without reading the implementation. If a test has 15 assertions covering 5 different behaviors, split it into 5 tests.

## The Arrange-Act-Assert (AAA) Pattern

Every unit test should follow three distinct phases:

1. **Arrange** — set up the test data, create objects, configure the scenario
2. **Act** — call the function or method under test (usually a single line)
3. **Assert** — verify the outcome matches expectations

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

The AAA pattern makes tests scannable. A reviewer can glance at any test and immediately understand the scenario, the action, and the expected result. Avoid mixing phases — do not assert in the middle of arrangement, and do not perform multiple unrelated actions.

**Anti-pattern: the "arrange-act-assert-act-assert" test.** If you find yourself acting and asserting multiple times in one test, you are testing multiple behaviors. Split it.

## Test Naming

Test names are documentation. When a test fails in CI, the name is often the first (and sometimes only) thing a developer reads. Good names describe the behavior being tested, not the method name.

**Effective naming patterns:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Pattern: <scenario>_<expected_result>
    #[test]
    fn empty_cart_has_zero_total() { /* ... */ }

    #[test]
    fn expired_coupon_is_rejected() { /* ... */ }

    #[test]
    fn negative_quantity_returns_validation_error() { /* ... */ }

    // Anti-pattern: meaningless names
    #[test]
    fn test1() { /* ... */ }  // What does this test?

    #[test]
    fn test_calculate_discount() { /* ... */ }  // Which behavior of calculate_discount?
}
```

When a test named `expired_coupon_is_rejected` fails, you know the coupon expiration logic is broken. When `test_calculate_discount_3` fails, you have to read the test body to figure out what went wrong.

## Rust Testing Mechanics

### The `#[test]` Attribute and `#[cfg(test)]`

Rust has first-class testing support. The `#[test]` attribute marks a function as a test. The `#[cfg(test)]` attribute on a module ensures the test code is only compiled when running `cargo test` — it is stripped from release builds.

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

### Assert Macros

Rust provides several built-in assert macros:

| Macro | Purpose | Example |
|-------|---------|---------|
| `assert!` | Asserts a boolean is true | `assert!(result.is_ok())` |
| `assert_eq!` | Asserts two values are equal | `assert_eq!(total, 42.0)` |
| `assert_ne!` | Asserts two values are not equal | `assert_ne!(id, 0)` |

All three accept an optional message as a trailing argument:

```rust
#[test]
fn user_age_must_be_positive() {
    let result = validate_age(-5);
    assert!(
        result.is_err(),
        "Expected validation error for negative age, got: {:?}",
        result
    );
}
```

Custom assertion messages are invaluable when tests fail in CI. The default `assertion failed: result.is_err()` tells you nothing about *why*. A descriptive message saves debugging time.

### Testing with `Result` Return Types

Tests can return `Result<(), E>` instead of panicking, enabling the `?` operator:

```rust
#[test]
fn parse_config_from_valid_toml() -> Result<(), Box<dyn std::error::Error>> {
    let toml = r#"
        [server]
        port = 8080
        host = "localhost"
    "#;

    let config = Config::from_toml(toml)?;
    assert_eq!(config.server.port, 8080);
    assert_eq!(config.server.host, "localhost");
    Ok(())
}
```

### `#[should_panic]` for Expected Failures

When a function is supposed to panic on invalid input, use `#[should_panic]`:

```rust
#[test]
#[should_panic(expected = "index out of bounds")]
fn accessing_beyond_length_panics() {
    let buffer = FixedBuffer::new(10);
    buffer.get(15);  // Should panic
}
```

Always include the `expected` string to ensure the test fails for the right reason. Without it, any panic passes the test — including panics from unrelated bugs.

## Real-World: Google's Testing Culture

Google expects engineers to write tests for every code change. Their internal code review tool, Critique, flags changes that lack tests. They categorize tests by size:

- **Small tests** (unit): must run in under 1 second, no I/O, no network, single process
- **Medium tests** (integration): must run in under 5 minutes, can use localhost network
- **Large tests** (E2E): must run in under 15 minutes, can use external systems

The vast majority of Google's tests are small tests. They have found that the return on investment of testing peaks at the unit level — these are the cheapest tests to write, the fastest to run, and the most precise when they fail. Their internal data shows that teams with strong unit test suites ship faster and have fewer production incidents.

Google enforces test quality through tooling: their build system (Blaze/Bazel) caches test results and only re-runs tests affected by a change. If your small tests are slow or flaky, the tooling flags it and your team lead notices.

## Common Pitfalls

### Testing Implementation Instead of Behavior

Tests should verify *what* code does, not *how* it does it internally. If you refactor a function's internals without changing its behavior, no tests should break.

```rust
// Bad: testing implementation detail (internal data structure order)
#[test]
fn cache_stores_items_in_insertion_order() {
    let mut cache = Cache::new();
    cache.insert("a", 1);
    cache.insert("b", 2);
    // This breaks if we switch from Vec to HashMap internally
    assert_eq!(cache.internal_items()[0].key, "a");
}

// Good: testing observable behavior
#[test]
fn cached_item_can_be_retrieved() {
    let mut cache = Cache::new();
    cache.insert("a", 1);
    assert_eq!(cache.get("a"), Some(&1));
}
```

### Testing Trivial Code

Do not write tests for simple getters, setters, or constructors with no logic. Focus testing effort on code with conditionals, calculations, and error handling. A test for `fn name(&self) -> &str { &self.name }` adds maintenance cost without catching any conceivable bug.

### Floating-Point Comparisons

Never use `assert_eq!` for floating-point results. Use epsilon-based comparison:

```rust
// Wrong: may fail due to floating-point representation
assert_eq!(0.1 + 0.2, 0.3);

// Correct: epsilon comparison
assert!((0.1_f64 + 0.2 - 0.3).abs() < f64::EPSILON);
```

## Key Takeaways

1. Good unit tests are fast, isolated, deterministic, and focused. Violation of any one of these makes the test unreliable.
2. Follow the AAA pattern (Arrange-Act-Assert) for consistent, readable tests.
3. Name tests after the behavior they verify, not the function they call.
4. Use `#[cfg(test)]` modules to keep test code out of production builds.
5. Write custom assertion messages — your future self debugging a CI failure will thank you.
6. Test behavior, not implementation. Tests should survive refactoring.
