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

```text
TEST deactivated_user_cannot_place_order
    // Arrange — set up the test data
    user ← User { id: 1, active: false }
    order ← NEW Order([NEW Item("Widget", 9.99)])

    // Act — perform the action being tested
    result ← PLACE_ORDER(user, order)

    // Assert — verify the expected outcome
    ASSERT result IS error
    ASSERT result.error = OrderError::InactiveUser
```

The AAA pattern makes tests scannable. A reviewer can glance at any test and immediately understand the scenario, the action, and the expected result. Avoid mixing phases — do not assert in the middle of arrangement, and do not perform multiple unrelated actions.

**Anti-pattern: the "arrange-act-assert-act-assert" test.** If you find yourself acting and asserting multiple times in one test, you are testing multiple behaviors. Split it.

## Test Naming

Test names are documentation. When a test fails in CI, the name is often the first (and sometimes only) thing a developer reads. Good names describe the behavior being tested, not the method name.

**Effective naming patterns:**

```text
TEST MODULE (only compiled during testing)

    // Pattern: <scenario>_<expected_result>
    TEST empty_cart_has_zero_total
        // ...

    TEST expired_coupon_is_rejected
        // ...

    TEST negative_quantity_returns_validation_error
        // ...

    // Anti-pattern: meaningless names
    TEST test1
        // ...    // What does this test?

    TEST test_calculate_discount
        // ...    // Which behavior of calculate_discount?
```

When a test named `expired_coupon_is_rejected` fails, you know the coupon expiration logic is broken. When `test_calculate_discount_3` fails, you have to read the test body to figure out what went wrong.

## Rust Testing Mechanics

### The `#[test]` Attribute and `#[cfg(test)]`

Rust has first-class testing support. The `#[test]` attribute marks a function as a test. The `#[cfg(test)]` attribute on a module ensures the test code is only compiled when running `cargo test` — it is stripped from release builds.

```text
// src/pricing
FUNCTION CALCULATE_DISCOUNT(price: float, discount_percent: float) → float
    IF discount_percent < 0.0 OR discount_percent > 100.0
        RETURN price
    RETURN price * (1.0 - discount_percent / 100.0)

TEST MODULE (only compiled during testing)

    TEST applies_percentage_discount
        result ← CALCULATE_DISCOUNT(100.0, 20.0)
        ASSERT |result - 80.0| < EPSILON

    TEST zero_discount_returns_original_price
        result ← CALCULATE_DISCOUNT(50.0, 0.0)
        ASSERT |result - 50.0| < EPSILON

    TEST full_discount_returns_zero
        result ← CALCULATE_DISCOUNT(100.0, 100.0)
        ASSERT |result - 0.0| < EPSILON

    TEST negative_discount_returns_original_price
        result ← CALCULATE_DISCOUNT(100.0, -10.0)
        ASSERT |result - 100.0| < EPSILON

    TEST discount_over_100_returns_original_price
        result ← CALCULATE_DISCOUNT(100.0, 150.0)
        ASSERT |result - 100.0| < EPSILON
```

### Assert Macros

Rust provides several built-in assert macros:

| Macro | Purpose | Example |
|-------|---------|---------|
| `assert!` | Asserts a boolean is true | `assert!(result.is_ok())` |
| `assert_eq!` | Asserts two values are equal | `assert_eq!(total, 42.0)` |
| `assert_ne!` | Asserts two values are not equal | `assert_ne!(id, 0)` |

All three accept an optional message as a trailing argument:

```text
TEST user_age_must_be_positive
    result ← VALIDATE_AGE(-5)
    ASSERT result IS error,
        "Expected validation error for negative age, got: " + result
```

Custom assertion messages are invaluable when tests fail in CI. The default `assertion failed: result.is_err()` tells you nothing about *why*. A descriptive message saves debugging time.

### Testing with `Result` Return Types

Tests can return `Result<(), E>` instead of panicking, enabling the `?` operator:

```text
TEST parse_config_from_valid_toml → Ok or Error
    toml ← "[server]
             port = 8080
             host = \"localhost\""

    config ← CONFIG_FROM_TOML(toml)?
    ASSERT config.server.port = 8080
    ASSERT config.server.host = "localhost"
    RETURN Ok
```

### `#[should_panic]` for Expected Failures

When a function is supposed to panic on invalid input, use `#[should_panic]`:

```text
TEST accessing_beyond_length_panics [EXPECT PANIC: "index out of bounds"]
    buffer ← NEW FixedBuffer(10)
    buffer.GET(15)  // Should panic
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

```text
// Bad: testing implementation detail (internal data structure order)
TEST cache_stores_items_in_insertion_order
    cache ← NEW Cache()
    cache.INSERT("a", 1)
    cache.INSERT("b", 2)
    // This breaks if we switch from Vec to HashMap internally
    ASSERT cache.INTERNAL_ITEMS()[0].key = "a"

// Good: testing observable behavior
TEST cached_item_can_be_retrieved
    cache ← NEW Cache()
    cache.INSERT("a", 1)
    ASSERT cache.GET("a") = Some(1)
```

### Testing Trivial Code

Do not write tests for simple getters, setters, or constructors with no logic. Focus testing effort on code with conditionals, calculations, and error handling. A test for `fn name(&self) -> &str { &self.name }` adds maintenance cost without catching any conceivable bug.

### Floating-Point Comparisons

Never use `assert_eq!` for floating-point results. Use epsilon-based comparison:

```text
// Wrong: may fail due to floating-point representation
ASSERT 0.1 + 0.2 = 0.3

// Correct: epsilon comparison
ASSERT |0.1 + 0.2 - 0.3| < EPSILON
```

## Key Takeaways

1. Good unit tests are fast, isolated, deterministic, and focused. Violation of any one of these makes the test unreliable.
2. Follow the AAA pattern (Arrange-Act-Assert) for consistent, readable tests.
3. Name tests after the behavior they verify, not the function they call.
4. Use `#[cfg(test)]` modules to keep test code out of production builds.
5. Write custom assertion messages — your future self debugging a CI failure will thank you.
6. Test behavior, not implementation. Tests should survive refactoring.
