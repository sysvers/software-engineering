# Code Coverage and Metrics

## What Code Coverage Measures

Code coverage answers a simple question: which parts of your code were executed during testing? It is a measure of *test reach*, not test quality. Coverage tools instrument your code and track which lines, branches, and functions are hit when the test suite runs.

Coverage is useful as a negative indicator: low coverage definitely means undertested code. But as a positive indicator, it is unreliable: high coverage does not mean the code is well tested.

## Types of Coverage

### Line Coverage

The most common metric. It counts the percentage of source lines executed by tests.

```rust
fn categorize_age(age: u32) -> &'static str {
    if age < 13 {          // Line 1
        "child"            // Line 2
    } else if age < 18 {   // Line 3
        "teenager"         // Line 4
    } else {               // Line 5
        "adult"            // Line 6
    }
}

#[test]
fn test_child() {
    assert_eq!(categorize_age(5), "child");
}
```

This single test hits lines 1 and 2 but misses lines 3-6. Line coverage: 33%. Adding a test for `categorize_age(15)` hits lines 3-4. Adding `categorize_age(25)` hits lines 5-6. Now coverage is 100%.

### Branch Coverage

Branch coverage tracks whether each decision point (if/else, match arms, boolean conditions) has been evaluated to both true and false. It is stricter than line coverage.

```rust
fn apply_discount(price: f64, is_member: bool, has_coupon: bool) -> f64 {
    if is_member && has_coupon {
        price * 0.7  // 30% off
    } else if is_member {
        price * 0.85 // 15% off
    } else if has_coupon {
        price * 0.9  // 10% off
    } else {
        price
    }
}
```

Line coverage might show 100% with three tests, but branch coverage requires testing all four paths. The combination `is_member=true, has_coupon=false` is a distinct branch from `is_member=true, has_coupon=true`.

### Function Coverage

The simplest metric: what percentage of functions were called at all? If you have 100 functions and tests call 80 of them, function coverage is 80%. This is a coarse metric — a function could be called once, exercising only its happy path, and it would count as covered.

## The Coverage Trap

This is the most important section in this document. Teams that chase coverage numbers often end up with worse test suites than teams that focus on meaningful tests.

### 100% Coverage with Zero Value

```rust
#[test]
fn covers_everything_tests_nothing() {
    let _ = process_payment(100.0, "USD");     // No assertions
    let _ = validate_email("test@example.com"); // No assertions
    let _ = calculate_shipping(5.0, "US");     // No assertions
}
```

This test achieves high coverage by calling functions and ignoring the results. It will never fail, which means it will never catch a bug. Coverage tools cannot tell the difference between a test that makes thoughtful assertions and a test that discards every return value.

### Coverage That Tests the Wrong Thing

```rust
#[test]
fn test_with_assertions_but_wrong_focus() {
    let result = calculate_tax(100.0, "CA");
    assert!(result > 0.0);  // Passes for any positive number
}
```

This test hits the line, asserts *something*, and achieves coverage. But `calculate_tax` returning `1.0` or `99.0` would both pass. The test does not verify correctness.

### The Goodhart Problem

Goodhart's Law: "When a measure becomes a target, it ceases to be a good measure." When teams are measured by coverage percentage, they optimize for the metric rather than test quality. They write tests that touch lines without verifying behavior. The coverage number goes up while the defect rate stays the same.

## Useful Coverage Targets

### The 70-80% Sweet Spot

For most projects, 70-80% line coverage indicates a healthy test suite. This range means:
- Core business logic is well tested
- Major code paths are exercised
- Error handling has reasonable coverage
- Some glue code and boilerplate is untested (which is fine)

### Where to Push Higher

- **Financial calculations** — 90%+ coverage is warranted. Off-by-one errors in monetary calculations have real consequences.
- **Security-sensitive code** — authentication, authorization, input validation. Test every path.
- **Public library APIs** — users depend on documented behavior. Every public function should be tested.

### Where to Accept Lower Coverage

- **Generated code** — auto-generated serialization, ORM boilerplate
- **Thin wrappers** — a function that just calls another function with slightly different arguments
- **Panic-only error paths** — `unreachable!()` and `unwrap()` on values you have already validated

## Tools for Rust

### cargo-tarpaulin

The most widely used coverage tool for Rust. It instruments your code and produces coverage reports.

```bash
# Install
cargo install cargo-tarpaulin

# Run with default settings
cargo tarpaulin

# Generate HTML report
cargo tarpaulin --out html

# Exclude test code from coverage metrics
cargo tarpaulin --ignore-tests

# Set a minimum coverage threshold (fail CI if below)
cargo tarpaulin --fail-under 70
```

Tarpaulin supports line and branch coverage on Linux. On macOS and Windows, it works via LLVM instrumentation but with some limitations. For CI pipelines, the `--fail-under` flag is useful as a ratchet: prevent coverage from dropping below the current level without blocking new code that does not need tests.

### cargo-nextest

Nextest is not a coverage tool — it is a faster test runner. It replaces `cargo test` and offers significantly better performance for large test suites through parallel execution, better scheduling, and improved output formatting.

```bash
# Install
cargo install cargo-nextest

# Run all tests
cargo nextest run

# Run with retries for flaky tests (identify them, then fix them)
cargo nextest run --retries 2

# Filter tests by name
cargo nextest run --filter-expr 'test(integration)'

# Generate JUnit XML for CI
cargo nextest run --profile ci
```

Nextest and tarpaulin complement each other. Use nextest for day-to-day test runs (fast feedback), and tarpaulin for periodic coverage measurement (CI pipeline).

### Combining the Tools

A typical CI pipeline:

```yaml
# .github/workflows/test.yml
jobs:
  test:
    steps:
      - name: Run tests
        run: cargo nextest run

      - name: Check coverage
        run: cargo tarpaulin --fail-under 70 --out xml

      - name: Upload coverage
        uses: codecov/codecov-action@v4
```

## The 80% Meaningful vs. 100% Superficial Principle

Two codebases:

**Codebase A: 100% coverage**
- Tests call every function but many assertions are weak (`assert!(result.is_ok())`)
- Mock behavior drifts from real dependencies
- Tests break on every refactor because they test implementation details
- Developers spend 40% of their time maintaining tests
- Production bugs still occur regularly

**Codebase B: 78% coverage**
- Uncovered code is mostly boilerplate and generated code
- Every test has specific, meaningful assertions
- Tests verify behavior through the public interface and survive refactoring
- Integration tests cover database queries and API contracts
- Developers trust the test suite and refactor confidently

Codebase B is better tested despite lower coverage. The missing 22% is code that is either trivial or better verified through integration and E2E tests. The 78% that *is* covered is tested thoroughly.

## Using Coverage as a Diagnostic Tool

The most productive way to use coverage is not as a scorecard but as a diagnostic tool:

1. **Find untested code paths.** Run coverage, look at uncovered lines. Are any of them important? If yes, write tests. If they are boilerplate, leave them.
2. **Identify dead code.** Code that is never executed in tests *and* never executed in production is dead. Remove it.
3. **Evaluate PR quality.** A code review tool that shows coverage diff can flag when a PR adds untested logic. This is more useful than an absolute coverage number.
4. **Prevent regression.** Use `--fail-under` in CI to prevent coverage from dropping. Ratchet it up gradually as the team writes more tests.

## Key Takeaways

1. Coverage measures test reach, not test quality. A test without assertions achieves coverage but catches nothing.
2. Line coverage is the most common metric, but branch coverage is more thorough. Function coverage is too coarse to be useful alone.
3. Aim for 70-80% line coverage in most projects. Push higher for financial, security, and public API code. Accept lower for generated code and boilerplate.
4. 80% meaningful coverage beats 100% superficial coverage every time. Focus on assertion quality, not line-touching.
5. Use cargo-tarpaulin for coverage measurement and cargo-nextest for faster test execution. Combine them in CI.
6. Treat coverage as a diagnostic tool for finding gaps, not as a target to optimize toward. When coverage becomes a goal, teams game it at the expense of real test quality.
