# Mutation Testing

## The Problem: Code Coverage Lies

Code coverage tells you which lines your tests execute. It does **not** tell you whether your tests actually verify anything meaningful.

```text
FUNCTION CALCULATE_DISCOUNT(price: float, is_member: boolean) -> float
    IF is_member
        RETURN price * 0.9  // 10% discount
    ELSE
        RETURN price

TEST test_discount
    // This test executes both branches -> 100% coverage
    CALL CALCULATE_DISCOUNT(100.0, true)
    CALL CALCULATE_DISCOUNT(100.0, false)
    // But it asserts NOTHING. It would pass even if the function returned 0.
```

100% code coverage. 0% bug detection. This is the gap mutation testing fills.

---

## How Mutation Testing Works

Mutation testing measures the **quality** of your tests by introducing small, deliberate bugs (mutations) into your code and checking if your tests catch them.

### The Process

1. Start with a passing test suite
2. Create a **mutant** — a copy of the code with one small change
3. Run the test suite against the mutant
4. If tests **fail** → the mutant is **killed** → your tests detected the bug (good)
5. If tests **pass** → the mutant **survived** → your tests missed a potential bug (bad)
6. Repeat with hundreds or thousands of mutations

### Types of Mutations

| Mutation Type | Original | Mutated | What It Reveals |
|--------------|----------|---------|-----------------|
| **Boundary** | `if age >= 18` | `if age > 18` | Missing boundary test (age = 18) |
| **Arithmetic** | `total + tax` | `total - tax` | Missing calculation verification |
| **Negation** | `if is_valid` | `if !is_valid` | Missing positive/negative path test |
| **Return value** | `return Ok(result)` | `return Err(error)` | Missing success path assertion |
| **Deletion** | `send_email(user)` | *(line removed)* | Missing side-effect verification |
| **Constant** | `timeout = 30` | `timeout = 0` | Missing timeout behavior test |
| **Comparison** | `a == b` | `a != b` | Missing equality check |

### Mutation Score

```
Mutation Score = Killed Mutants / Total Mutants
```

- **90%+** — excellent test quality, thorough assertions
- **70-90%** — good, but some gaps worth investigating
- **Below 70%** — significant gaps in test effectiveness

---

## Mutation Testing in Rust with cargo-mutants

[cargo-mutants](https://github.com/sourcefrog/cargo-mutants) is the primary mutation testing tool for Rust. It generates mutations, runs your test suite against each one, and reports which survived.

### Basic Usage

```bash
# Install
cargo install cargo-mutants

# Run against your project
cargo mutants

# Run with parallel jobs for speed
cargo mutants -j 4

# Test only specific files
cargo mutants --file src/pricing.rs
```

### Example: Finding Weak Tests

Consider this Rust module:

```text
// src/pricing
FUNCTION APPLY_DISCOUNT(price: float, discount_percent: float) -> float
    IF discount_percent < 0.0 OR discount_percent > 100.0
        RETURN price  // Invalid discount, return original price
    RETURN price * (1.0 - discount_percent / 100.0)

FUNCTION CALCULATE_TOTAL(items: list of (price: float, qty: unsigned integer)) -> float
    RETURN SUM OF (price * qty) FOR EACH (price, qty) IN items
```

And these tests:

```text
TEST MODULE

    TEST test_apply_discount
        result <- APPLY_DISCOUNT(100.0, 10.0)
        ASSERT result = 90.0

    TEST test_calculate_total
        items <- [(10.0, 2), (5.0, 3)]
        total <- CALCULATE_TOTAL(items)
        ASSERT total = 35.0
```

Running `cargo mutants` would generate mutations like:

```
src/pricing.rs: replace apply_discount → 0.0          ... SURVIVED
src/pricing.rs: replace < with <= in discount_percent  ... SURVIVED
src/pricing.rs: replace > with >= in discount_percent  ... SURVIVED
src/pricing.rs: replace - with + in price calculation  ... KILLED
src/pricing.rs: replace calculate_total → 0.0          ... KILLED
```

Three mutants survived, revealing:

1. **No test for invalid discount** — replacing the function with `return 0.0` passes because no test checks the boundary validation
2. **No boundary test at 0%** — changing `< 0.0` to `<= 0.0` passes because no test uses `discount_percent = 0.0`
3. **No boundary test at 100%** — changing `> 100.0` to `>= 100.0` passes because no test uses `discount_percent = 100.0`

### Fixing the Tests

```text
TEST MODULE

    TEST test_apply_discount_normal
        ASSERT APPLY_DISCOUNT(100.0, 10.0) = 90.0

    TEST test_apply_discount_zero_percent
        ASSERT APPLY_DISCOUNT(100.0, 0.0) = 100.0  // Kills boundary mutant

    TEST test_apply_discount_full
        ASSERT APPLY_DISCOUNT(100.0, 100.0) = 0.0  // Kills boundary mutant

    TEST test_apply_discount_negative_rejected
        ASSERT APPLY_DISCOUNT(100.0, -5.0) = 100.0  // Kills validation mutant

    TEST test_apply_discount_over_100_rejected
        ASSERT APPLY_DISCOUNT(100.0, 150.0) = 100.0  // Kills validation mutant

    TEST test_calculate_total
        items <- [(10.0, 2), (5.0, 3)]
        ASSERT CALCULATE_TOTAL(items) = 35.0

    TEST test_calculate_total_empty
        ASSERT CALCULATE_TOTAL([]) = 0.0
```

Now `cargo mutants` kills all mutants — the tests actually verify the code's behavior.

---

## Real Bugs Found by Mutation Testing

### Off-by-One in Pagination

```text
// Original: items per page = 20
FUNCTION PAGINATE(items: list of Item, page: size) -> slice of items
    start <- page * 20
    end <- MIN(start + 20, items.length)
    RETURN items[start..end]
```

Mutation testing changed `page * 20` to `page * 20 + 1`. Tests still passed because no test checked the exact boundary between page 0 and page 1. The test was only checking "does page 0 return some items?" rather than "does page 0 return exactly the first 20 items?"

### Silent Failure in Error Handling

```text
FUNCTION PROCESS_PAYMENT(order: Order) -> Receipt or PaymentError
    charge <- gateway.CHARGE(order.total)?
    AUDIT_LOG.RECORD_PAYMENT(charge)  // Side effect
    RETURN Ok(Receipt from charge)
```

Mutation testing deleted the `audit_log::record_payment` line. Tests still passed — no test verified that the audit log was written. In production, this would mean payments processed without audit trails, a compliance violation.

### Wrong Comparison Operator

```text
FUNCTION IS_ELIGIBLE_FOR_FREE_SHIPPING(order_total: float) -> boolean
    RETURN order_total ≥ 50.0  // Free shipping on orders $50+
```

Mutation testing changed `>=` to `>`. The test suite used `order_total = 100.0` — far from the boundary. It never tested the exact threshold of `$50.00`, so the mutant survived. A customer ordering exactly $50 worth of items would have been incorrectly charged for shipping.

---

## Practical Considerations

### Performance

Mutation testing is slow. Each mutant requires a full test suite run. For a project with 500 mutants and a 30-second test suite, that's over 4 hours.

**Strategies to manage runtime:**

- **Run on specific files** — focus on recently changed code: `cargo mutants --file src/changed_module.rs`
- **Use incremental mode** — only test new/changed mutants
- **Parallelize** — `cargo mutants -j 8` runs multiple mutants concurrently
- **Run in CI nightly**, not on every PR — treat it as a periodic quality check

### Equivalent Mutants

Some mutations produce code that is functionally identical to the original. These are **equivalent mutants** — they survive not because tests are weak, but because the mutation doesn't change behavior.

```text
// Original
items <- COLLECT(data.ITERATE())

// Mutant: replace collect with collect (same thing via different path)
// This is equivalent -- it's not a test gap
```

Equivalent mutants inflate the "survived" count. Manually review survived mutants to distinguish real gaps from equivalents.

### What to Do with Survived Mutants

1. **Is it an equivalent mutant?** If the mutation doesn't change observable behavior, ignore it.
2. **Is the untested behavior important?** Not every code path needs testing. A log message format change might not warrant a test.
3. **If it's important, write the test.** The survived mutant tells you exactly what assertion is missing.

---

## When to Use Mutation Testing

**Use for:**
- Critical business logic (pricing, permissions, validation)
- Security-sensitive code (authentication, authorization)
- Code where "tests pass but bugs ship" is a recurring problem
- Evaluating test quality beyond coverage metrics

**Avoid for:**
- UI code (mutations on CSS or layout are meaningless)
- Generated code (test the generator, not the output)
- Prototype or throwaway code
- As a blocking CI gate on every PR (too slow for most projects)

---

## Key Takeaways

1. Code coverage measures execution, not verification. Mutation testing measures whether tests actually catch bugs.
2. A mutation score of 90%+ indicates thorough, well-asserted tests.
3. Survived mutants point directly to specific gaps — missing boundary tests, unverified side effects, weak assertions.
4. Run mutation testing on critical code paths periodically, not necessarily on every commit.
5. Always review survived mutants manually — some are equivalent mutants that don't represent real gaps.
