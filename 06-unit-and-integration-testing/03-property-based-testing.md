# Property-Based Testing

## The Limits of Example-Based Testing

Traditional tests use specific examples: "given input X, expect output Y." This works well for known cases, but it has a fundamental limitation — you can only test the cases you think of. Edge cases you do not imagine go untested.

Property-based testing takes a different approach. Instead of specifying examples, you specify *properties* — invariants that must hold for *all* inputs — and the framework generates hundreds or thousands of random inputs to try to violate those properties.

The technique was pioneered by QuickCheck in Haskell (1999) and has since been adopted across languages. In Rust, the primary library is `proptest`.

## Properties vs. Examples

An example-based test says: "sorting `[3, 1, 2]` should produce `[1, 2, 3]`."

A property-based test says: "for *any* input list, the sorted output should be (1) the same length, (2) in non-decreasing order, and (3) a permutation of the input."

```text
// Example-based: specific case
TEST sort_works_for_specific_input
    ASSERT SORT([3, 1, 2]) = [1, 2, 3]

// Property-based: any input
PROPERTY TESTS:

    TEST sorted_output_has_same_length(input: list of integers)
        sorted ← SORT(CLONE(input))
        ASSERT sorted.length = input.length

    TEST sorted_output_is_ordered(input: list of integers)
        sorted ← SORT(input)
        FOR EACH consecutive pair (a, b) IN sorted
            ASSERT a ≤ b

    TEST sorted_output_contains_same_elements(input: list of integers)
        sorted ← SORT(CLONE(input))
        original ← SORT(input)
        ASSERT sorted = original
```

The property-based tests will exercise empty vectors, single-element vectors, already-sorted vectors, reverse-sorted vectors, vectors with duplicates, vectors with `i32::MIN` and `i32::MAX`, and thousands of other cases you would never write by hand.

## Using `proptest` in Rust

### Setup

Add to your `Cargo.toml`:

```toml
[dev-dependencies]
proptest = "1"
```

### Basic Usage

The `proptest!` macro generates test cases from strategies. Built-in strategies exist for all primitive types, `String`, `Vec<T>`, `Option<T>`, `Result<T, E>`, and tuples.

```text
PROPERTY TESTS:

    TEST addition_is_commutative(a: integer, b: integer)
        ASSERT WRAPPING_ADD(a, b) = WRAPPING_ADD(b, a)

    TEST string_roundtrips_through_uppercase(s: string)
        // A property: uppercasing and lowercasing back should preserve length
        ASSERT UPPERCASE(s).length ≥ s.length
```

### Custom Strategies

When built-in strategies generate inputs that are too broad, narrow them with custom strategies:

```text
// Generate valid email-like strings
STRATEGY EMAIL_STRATEGY() → string matching "[a-z]{1,10}@[a-z]{1,10}.[a-z]{2,4}"

// Generate prices in a realistic range
STRATEGY PRICE_STRATEGY() → float in range 0.01 to 10000.0

// Generate vectors with constrained length
STRATEGY SHORT_VEC_STRATEGY() → list of integers with length 0 to 100

PROPERTY TESTS:

    TEST discount_never_produces_negative_price(
        price FROM PRICE_STRATEGY,
        discount FROM range 0.0 to 100.0
    )
        result ← CALCULATE_DISCOUNT(price, discount)
        ASSERT result ≥ 0.0, "Negative price: " + price + " with " + discount + "% discount = " + result
```

### Composing Strategies

Build complex test data from simpler strategies:

```text
STRUCTURE Order
    customer_id: unsigned integer
    items: list of OrderItem
    coupon_code: optional string

STRUCTURE OrderItem
    name: string
    price: float
    quantity: unsigned integer

STRATEGY ORDER_ITEM_STRATEGY() → OrderItem
    name FROM random string matching "[a-zA-Z ]{1,50}"
    price FROM range 0.01 to 1000.0
    quantity FROM range 1 to 100
    RETURN OrderItem { name, price, quantity }

STRATEGY ORDER_STRATEGY() → Order
    customer_id FROM range 1 to 1000000
    items FROM list of ORDER_ITEM_STRATEGY, length 1 to 20
    coupon_code FROM optional string matching "[A-Z]{4,8}"
    RETURN Order { customer_id, items, coupon_code }

PROPERTY TESTS:

    TEST order_total_is_sum_of_item_totals(order FROM ORDER_STRATEGY)
        expected ← SUM OF (item.price * item.quantity) FOR EACH item IN order.items
        actual ← CALCULATE_ORDER_TOTAL(order)
        ASSERT |actual - expected| < 0.01
```

## Shrinking: Finding Minimal Failing Cases

When proptest finds a failing input, it does not just report the random input that failed. It *shrinks* the input — systematically reducing it to the smallest input that still triggers the failure. This is one of proptest's most valuable features.

For example, if your sort function fails on a vector of 47 elements, proptest will shrink it down to a 2-element vector that demonstrates the same bug. Instead of debugging with 47 elements, you debug with 2.

```text
PROPERTY TESTS:

    TEST buggy_dedup_preserves_all_unique_elements(input: list of integers)
        deduped ← BUGGY_DEDUP(CLONE(input))
        unique ← SET of all elements in input
        // If this fails on [1, 2, 1, 3, 2], proptest might shrink to [0, 0]
        ASSERT deduped.length = unique.size
```

Proptest's shrinking output looks like:

```
test buggy_dedup_preserves_all_unique_elements ... FAILED
minimal failing input: input = [0, 0]
```

This tells you immediately: the bug is triggered by duplicate elements. You do not need to reason about the original 47-element vector.

### Persistence of Failing Cases

Proptest saves failing cases to a file (`.proptest-regressions/`) so they are re-run on subsequent test runs. This ensures that once a bug is found, the specific failing input becomes a permanent regression test — even after you fix the bug.

## Real Bugs Found by Property-Based Testing

### Serialization Roundtrip Failures

One of the most common classes of bugs found by property testing: a value that does not survive serialization and deserialization.

```text
PROPERTY TESTS:

    TEST json_roundtrip_preserves_data(
        name FROM "[a-zA-Z ]{1,100}",
        age FROM range 0 to 200,
        score FROM any float
    )
        original ← UserProfile { name, age, score }
        json ← JSON_SERIALIZE(original)
        deserialized ← JSON_DESERIALIZE(json) as UserProfile
        ASSERT original.name = deserialized.name
        ASSERT original.age = deserialized.age
        // NaN != NaN, so special handling needed
        IF original.score IS NaN
            ASSERT deserialized.score IS NaN
        ELSE
            ASSERT |original.score - deserialized.score| < EPSILON
```

This test commonly finds bugs with special float values (NaN, infinity), Unicode edge cases in strings, and off-by-one errors in custom serialization.

### Encoder/Decoder Symmetry

```text
PROPERTY TESTS:

    TEST base64_roundtrip(data: list of bytes)
        encoded ← BASE64_ENCODE(data)
        decoded ← BASE64_DECODE(encoded)
        ASSERT data = decoded
```

A real bug found by this pattern: a base64 encoder that failed on inputs whose length was not a multiple of 3, producing incorrect padding.

### Arithmetic Overflow

```text
PROPERTY TESTS:

    TEST total_price_does_not_overflow(
        prices FROM list of floats in range 0.01 to 1000000.0, length 0 to 1000
    )
        total ← COMPUTE_TOTAL(prices)
        ASSERT total IS finite, "Overflow: total = " + total
        ASSERT total ≥ 0.0
```

### Idempotency

```text
PROPERTY TESTS:

    TEST formatting_is_idempotent(input FROM "[a-zA-Z0-9 .,!?]{0,200}")
        once ← NORMALIZE_WHITESPACE(input)
        twice ← NORMALIZE_WHITESPACE(once)
        ASSERT once = twice, "Formatting is not idempotent"
```

This found a bug where normalizing whitespace turned double spaces into single spaces but also converted tabs into double spaces, meaning a second pass would change the output again.

## Writing Good Properties

The hardest part of property-based testing is choosing which properties to test. Here are common property patterns:

### Roundtrip / Symmetry

If you encode then decode, you should get the original value back. Applies to serialization, compression, encryption, parsing, and formatting.

### Invariant Preservation

After any operation, certain invariants must hold. A sorted list stays sorted after insertion. A balanced tree stays balanced after deletion. A bank account balance equals deposits minus withdrawals.

### Equivalence to a Reference

Your optimized implementation should produce the same results as a simple (correct but slow) reference implementation.

```text
PROPERTY TESTS:

    TEST fast_median_matches_naive_median(input FROM list of floats, length 1 to 1000)
        fast ← FAST_MEDIAN(input)
        naive ← NAIVE_MEDIAN(input)
        ASSERT |fast - naive| < 0.001
```

### Commutativity, Associativity, Distributivity

Mathematical properties that your operations should satisfy, if applicable.

### "No Crash" Property

The weakest but still useful property: the function does not panic for any valid input.

```text
PROPERTY TESTS:

    TEST parse_never_panics(input: string)
        CALL PARSE_CONFIG(input)  // May return Err, but should never panic
```

## Practical Considerations

### Test Speed

Property-based tests are slower than example-based tests because they run many iterations (default: 256 cases per test). For expensive operations, reduce the case count:

```text
PROPERTY TESTS (config: 50 cases instead of default 256):

    TEST expensive_operation_property(input: list of integers)
        // Only 50 cases instead of 256
        result ← EXPENSIVE_OPERATION(input)
        ASSERT result.IS_VALID()
```

### Debugging Failures

When a property test fails, proptest prints the minimal failing input. Reproduce it with a targeted example-based test:

```text
// proptest found: input = [0, 0]
TEST regression_dedup_with_duplicates
    result ← BUGGY_DEDUP([0, 0])
    ASSERT result = [0]
```

This example-based regression test is faster to run and easier to debug than the property test.

### When NOT to Use Property-Based Testing

- **Simple CRUD operations** with no interesting invariants
- **UI rendering** where properties are hard to express
- **Tests that require specific expected outputs** rather than general invariants
- **Code where generating valid inputs is harder than writing examples** (complex protocol messages, deeply nested state machines)

## Key Takeaways

1. Property-based testing finds edge cases that example-based tests miss: empty inputs, boundary values, overflow, Unicode, and more.
2. Think in properties (invariants, roundtrips, equivalence) rather than specific examples.
3. Shrinking is proptest's killer feature — it reduces complex failing inputs to minimal reproducible cases.
4. Use custom strategies to generate realistic test data within your domain's constraints.
5. Property tests complement example tests; they do not replace them. Use examples for readability and known important cases, properties for broad coverage.
6. Start with the "no crash" property — it is the easiest to write and catches more bugs than you might expect.
