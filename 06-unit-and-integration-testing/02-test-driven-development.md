# Test-Driven Development (TDD)

## What Is TDD?

Test-Driven Development is a discipline where you write the test *before* you write the production code. It inverts the typical workflow: instead of "write code, then write tests," you do "write test, then write code." This is not just a testing technique — it is a design technique. The act of writing the test first forces you to think about the interface before the implementation.

TDD was popularized by Kent Beck in the early 2000s and became a cornerstone of Extreme Programming (XP). It remains one of the most debated practices in software engineering — not because it does not work, but because it requires discipline and is easy to do poorly.

## The Red-Green-Refactor Cycle

TDD follows a tight, repeating cycle with three phases:

### 1. Red — Write a Failing Test

Write a test for the behavior you want to add. This test should fail — either by not compiling (in Rust, this is common) or by producing a wrong result. The failure is important: it proves the test is actually testing something. A test that passes immediately might be testing nothing.

### 2. Green — Write the Minimum Code to Pass

Write the simplest possible production code that makes the test pass. Do not optimize. Do not generalize. Do not handle edge cases you have not tested yet. The goal is to go from red to green as quickly as possible.

This step feels unnatural at first. You might write code you *know* is incomplete. That is the point. The incompleteness will be addressed by the next test.

### 3. Refactor — Clean Up While Staying Green

Now improve the code's structure, remove duplication, improve naming — while keeping all tests passing. The test suite acts as a safety net during refactoring. If you break something, you find out immediately.

```
Write failing test --> Write code to pass --> Refactor --> Repeat
      (Red)               (Green)            (Clean)
```

## Step-by-Step TDD in Rust: Building a Shopping Cart

Let us build a shopping cart from scratch using strict TDD. Every line of production code will be motivated by a failing test.

### Cycle 1: Empty Cart Has Zero Total

**Red** — write the test first:

```text
// src/cart
TEST MODULE

    TEST empty_cart_has_zero_total
        cart ← NEW ShoppingCart()
        ASSERT cart.TOTAL() = 0.0
```

This will not compile because `ShoppingCart` does not exist.

**Green** — write the minimum code:

```text
STRUCTURE ShoppingCart

    FUNCTION NEW() → ShoppingCart
        RETURN ShoppingCart

    FUNCTION TOTAL(self) → float
        RETURN 0.0
```

The test passes. Yes, `total()` is hardcoded to return 0.0. That is fine — the next test will force us to make it real.

**Refactor** — nothing to clean up yet.

### Cycle 2: Cart With One Item

**Red** — next test:

```text
TEST cart_with_one_item_returns_item_price
    cart ← NEW ShoppingCart()
    cart.ADD_ITEM("Widget", 9.99)
    ASSERT |cart.TOTAL() - 9.99| < EPSILON
```

Does not compile: `add_item` does not exist, and `ShoppingCart` is a unit struct with no fields.

**Green** — extend the implementation:

```text
STRUCTURE ShoppingCart
    items: list of (name: string, price: float)

    FUNCTION NEW() → ShoppingCart
        RETURN ShoppingCart { items: empty list }

    FUNCTION ADD_ITEM(self, name: string, price: float)
        APPEND (name, price) TO self.items

    FUNCTION TOTAL(self) → float
        RETURN SUM OF price FOR EACH (_, price) IN self.items
```

Both tests pass.

**Refactor** — the code is clean enough for now.

### Cycle 3: Multiple Items

**Red**:

```text
TEST cart_with_multiple_items_sums_prices
    cart ← NEW ShoppingCart()
    cart.ADD_ITEM("Widget", 9.99)
    cart.ADD_ITEM("Gadget", 24.99)
    cart.ADD_ITEM("Doohickey", 5.00)

    expected ← 9.99 + 24.99 + 5.00
    ASSERT |cart.TOTAL() - expected| < 0.001
```

**Green** — this already passes with our current implementation. When a test passes immediately, it means either (a) the behavior was already covered, or (b) the test is not testing what you think. In this case, `iter().sum()` naturally handles multiple items. The test still has value as a regression guard.

### Cycle 4: Removing Items

**Red**:

```text
TEST removing_item_reduces_total
    cart ← NEW ShoppingCart()
    cart.ADD_ITEM("Widget", 9.99)
    cart.ADD_ITEM("Gadget", 24.99)
    cart.REMOVE_ITEM("Widget")

    ASSERT |cart.TOTAL() - 24.99| < EPSILON
```

**Green**:

```text
FUNCTION REMOVE_ITEM(self, name: string)
    pos ← FIND INDEX in self.items WHERE item.name = name
    IF pos IS found
        REMOVE self.items[pos]
```

### Cycle 5: Applying a Discount

**Red**:

```text
TEST discount_reduces_total_by_percentage
    cart ← NEW ShoppingCart()
    cart.ADD_ITEM("Widget", 100.0)
    cart.APPLY_DISCOUNT(20.0)  // 20% off

    ASSERT |cart.TOTAL() - 80.0| < EPSILON
```

**Green**:

```text
STRUCTURE ShoppingCart
    items: list of (name: string, price: float)
    discount_percent: float

    FUNCTION NEW() → ShoppingCart
        RETURN ShoppingCart {
            items: empty list,
            discount_percent: 0.0
        }

    FUNCTION APPLY_DISCOUNT(self, percent: float)
        self.discount_percent ← percent

    FUNCTION TOTAL(self) → float
        subtotal ← SUM OF price FOR EACH (_, price) IN self.items
        RETURN subtotal * (1.0 - self.discount_percent / 100.0)
```

**Refactor** — all five tests pass. Notice how the design *emerged* from the tests. We did not plan the `discount_percent` field upfront; the test drove us to add it.

## When TDD Works Well

### Well-Defined Business Logic

TDD excels when the requirements are clear and expressible as input-output relationships. Pricing engines, validation rules, tax calculations, scoring algorithms — these are TDD's sweet spot. Each rule becomes a test, and the implementation naturally follows.

### Bug Fixes

TDD's most universally applicable use case: when fixing a bug, write a test that reproduces the bug first. This guarantees the bug is actually fixed (the test goes from red to green) and prevents regression (the test will catch the bug if it returns). Even teams that do not practice TDD for new features should adopt this pattern for bug fixes.

### Parsers and Transformers

Code that transforms one data format into another is naturally test-friendly. You know the input, you know the expected output. TDD here produces comprehensive test suites almost effortlessly.

```text
TEST parses_csv_line_into_record
    line ← "Alice,30,alice@example.com"
    record ← PARSE_CSV_LINE(line)
    ASSERT record.name = "Alice"
    ASSERT record.age = 30
    ASSERT record.email = "alice@example.com"

TEST handles_quoted_fields_with_commas
    line ← "\"Smith, Alice\",30,alice@example.com"
    record ← PARSE_CSV_LINE(line)
    ASSERT record.name = "Smith, Alice"
```

### Algorithm Development

Sorting, searching, graph traversal, dynamic programming — TDD forces you to enumerate the cases (empty input, single element, already sorted, reverse sorted, duplicates) before writing the algorithm. This thoroughness often reveals edge cases you would otherwise miss.

## When TDD Is Awkward

### Exploratory Prototyping

When you do not yet know what you are building — when you are sketching, trying different approaches, throwing away code — TDD adds friction without benefit. The tests you write will be thrown away with the prototype. In this case, explore first, then write tests once the design stabilizes.

### UI and Layout Code

Frontend code with rapidly changing layouts does not benefit from TDD. The cost of maintaining tests for visual layouts exceeds the value. Test the *logic* behind the UI (validation, state management, formatting), not the layout itself.

### Thin Glue Code

Code that mostly wires together other components — a controller that calls a service that calls a repository — has little logic to test. TDD here produces tests that are essentially re-statements of the implementation. Integration tests are more valuable for this layer.

### Performance-Critical Code

When you need to optimize for speed and are benchmarking different approaches, TDD's incremental style can slow you down. Write the tests after you have found the right algorithm and data structure.

## Real-World Example: TDD Saves a Financial Trading Platform

A financial services firm adopted TDD for their trading engine's settlement date calculator. When a regulatory change required different settlement date logic (accounting for market-specific holidays, weekends, and cross-timezone trades), they followed TDD strictly:

1. They wrote tests for every rule in the new regulation — standard weekday settlements, Friday trades settling on Monday, holiday handling across US, UK, and Hong Kong markets, cross-timezone edge cases.
2. They modified the implementation to pass each test, one at a time.
3. Edge cases that would have been missed (a holiday in one timezone but not another, a leap year falling on a weekend) were caught by the tests before the code was written.

The migration was completed in 2 days with zero production incidents. The previous regulatory change (done without TDD) had taken 2 weeks and produced 3 production bugs affecting settlement amounts. The difference was not the developers' skill — it was the discipline of enumerating cases before writing code.

## TDD Anti-Patterns

### Writing Too Many Tests Before Any Code

TDD is one test at a time. Writing 20 tests before any production code means you spend a long time in "red" and then make a massive jump to "green." This loses the incremental design benefit.

### Testing Implementation Details

If your test needs to check that a private method was called, or that an internal data structure has a specific layout, the test is too coupled to the implementation. TDD tests should verify observable behavior through the public interface.

### The "Make It Pass With a Constant" Trap

In strict TDD, the first green step might be returning a hardcoded value. This is fine as a stepping stone, but if you find yourself avoiding the real implementation across many cycles, you are gaming the process instead of using it.

### Abandoning TDD During Refactoring

The refactor step is not optional. Skipping it leads to code that works but accumulates technical debt. The test suite gives you permission to refactor aggressively — use it.

## Trade-offs Summary

| Factor | TDD | Test-After |
|--------|-----|------------|
| Initial development speed | Slower | Faster |
| Test coverage | Naturally high | Often patchy |
| Design quality | Drives clean interfaces | Risk of tightly-coupled code |
| Bug detection | Catches issues before code is written | Catches issues after code is written |
| Exploratory work | Awkward and constraining | Natural and flexible |
| Maintenance cost | Lower long-term (clean design) | Higher long-term (harder to test, harder to refactor) |

## Key Takeaways

1. TDD is Red-Green-Refactor: write a failing test, make it pass with minimal code, then clean up. One cycle at a time.
2. TDD is a design technique, not just a testing technique. Writing the test first forces you to design the interface before the implementation.
3. TDD works best for well-defined business logic, bug fixes, parsers, and algorithms. It is awkward for exploratory work, UI layouts, and thin glue code.
4. The strongest argument for TDD is regression prevention: every behavior has a test, so regressions are caught immediately.
5. TDD requires discipline. Done poorly (skipping refactoring, testing implementation details), it adds cost without benefit. Done well, it produces clean, well-tested code faster than the alternative.
