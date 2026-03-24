# Property-Based Testing

## The Limits of Example-Based Testing

Traditional tests use specific examples: "given input X, expect output Y." This works well for known cases, but it has a fundamental limitation — you can only test the cases you think of. Edge cases you do not imagine go untested.

Property-based testing takes a different approach. Instead of specifying examples, you specify *properties* — invariants that must hold for *all* inputs — and the framework generates hundreds or thousands of random inputs to try to violate those properties.

The technique was pioneered by QuickCheck in Haskell (1999) and has since been adopted across languages. In Rust, the primary library is `proptest`.

## Properties vs. Examples

An example-based test says: "sorting `[3, 1, 2]` should produce `[1, 2, 3]`."

A property-based test says: "for *any* input list, the sorted output should be (1) the same length, (2) in non-decreasing order, and (3) a permutation of the input."

```rust
// Example-based: specific case
#[test]
fn sort_works_for_specific_input() {
    assert_eq!(sort(vec![3, 1, 2]), vec![1, 2, 3]);
}

// Property-based: any input
use proptest::prelude::*;

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

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn addition_is_commutative(a: i64, b: i64) {
        assert_eq!(a.wrapping_add(b), b.wrapping_add(a));
    }

    #[test]
    fn string_roundtrips_through_uppercase(s: String) {
        // A property: uppercasing and lowercasing back should preserve length
        assert_eq!(s.to_uppercase().len() >= s.len(), true);
    }
}
```

### Custom Strategies

When built-in strategies generate inputs that are too broad, narrow them with custom strategies:

```rust
use proptest::prelude::*;

// Generate valid email-like strings
fn email_strategy() -> impl Strategy<Value = String> {
    ("[a-z]{1,10}@[a-z]{1,10}\\.[a-z]{2,4}").boxed()
}

// Generate prices in a realistic range
fn price_strategy() -> impl Strategy<Value = f64> {
    (0.01f64..10_000.0)
}

// Generate vectors with constrained length
fn short_vec_strategy() -> impl Strategy<Value = Vec<i32>> {
    prop::collection::vec(any::<i32>(), 0..100)
}

proptest! {
    #[test]
    fn discount_never_produces_negative_price(
        price in price_strategy(),
        discount in 0.0f64..100.0,
    ) {
        let result = calculate_discount(price, discount);
        assert!(result >= 0.0, "Negative price: {} with {}% discount = {}", price, discount, result);
    }
}
```

### Composing Strategies

Build complex test data from simpler strategies:

```rust
use proptest::prelude::*;

#[derive(Debug, Clone)]
struct Order {
    customer_id: u64,
    items: Vec<OrderItem>,
    coupon_code: Option<String>,
}

#[derive(Debug, Clone)]
struct OrderItem {
    name: String,
    price: f64,
    quantity: u32,
}

fn order_item_strategy() -> impl Strategy<Value = OrderItem> {
    (
        "[a-zA-Z ]{1,50}",       // name
        0.01f64..1000.0,          // price
        1u32..100,                // quantity
    )
        .prop_map(|(name, price, quantity)| OrderItem { name, price, quantity })
}

fn order_strategy() -> impl Strategy<Value = Order> {
    (
        1u64..1_000_000,                              // customer_id
        prop::collection::vec(order_item_strategy(), 1..20),  // 1-20 items
        prop::option::of("[A-Z]{4,8}"),               // optional coupon
    )
        .prop_map(|(customer_id, items, coupon_code)| Order {
            customer_id,
            items,
            coupon_code,
        })
}

proptest! {
    #[test]
    fn order_total_is_sum_of_item_totals(order in order_strategy()) {
        let expected: f64 = order.items.iter()
            .map(|item| item.price * item.quantity as f64)
            .sum();
        let actual = calculate_order_total(&order);
        assert!((actual - expected).abs() < 0.01);
    }
}
```

## Shrinking: Finding Minimal Failing Cases

When proptest finds a failing input, it does not just report the random input that failed. It *shrinks* the input — systematically reducing it to the smallest input that still triggers the failure. This is one of proptest's most valuable features.

For example, if your sort function fails on a vector of 47 elements, proptest will shrink it down to a 2-element vector that demonstrates the same bug. Instead of debugging with 47 elements, you debug with 2.

```rust
proptest! {
    #[test]
    fn buggy_dedup_preserves_all_unique_elements(input: Vec<i32>) {
        let deduped = buggy_dedup(input.clone());
        let unique: HashSet<i32> = input.iter().copied().collect();
        // If this fails on [1, 2, 1, 3, 2], proptest might shrink to [0, 0]
        assert_eq!(deduped.len(), unique.len());
    }
}
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

```rust
proptest! {
    #[test]
    fn json_roundtrip_preserves_data(
        name in "[a-zA-Z ]{1,100}",
        age in 0u32..200,
        score in any::<f64>(),
    ) {
        let original = UserProfile { name: name.clone(), age, score };
        let json = serde_json::to_string(&original).unwrap();
        let deserialized: UserProfile = serde_json::from_str(&json).unwrap();
        assert_eq!(original.name, deserialized.name);
        assert_eq!(original.age, deserialized.age);
        // NaN != NaN, so special handling needed
        if original.score.is_nan() {
            assert!(deserialized.score.is_nan());
        } else {
            assert!((original.score - deserialized.score).abs() < f64::EPSILON);
        }
    }
}
```

This test commonly finds bugs with special float values (NaN, infinity), Unicode edge cases in strings, and off-by-one errors in custom serialization.

### Encoder/Decoder Symmetry

```rust
proptest! {
    #[test]
    fn base64_roundtrip(data: Vec<u8>) {
        let encoded = base64_encode(&data);
        let decoded = base64_decode(&encoded).unwrap();
        assert_eq!(data, decoded);
    }
}
```

A real bug found by this pattern: a base64 encoder that failed on inputs whose length was not a multiple of 3, producing incorrect padding.

### Arithmetic Overflow

```rust
proptest! {
    #[test]
    fn total_price_does_not_overflow(
        prices in prop::collection::vec(0.01f64..1_000_000.0, 0..1000),
    ) {
        let total = compute_total(&prices);
        assert!(total.is_finite(), "Overflow: total = {}", total);
        assert!(total >= 0.0);
    }
}
```

### Idempotency

```rust
proptest! {
    #[test]
    fn formatting_is_idempotent(input in "[a-zA-Z0-9 .,!?]{0,200}") {
        let once = normalize_whitespace(&input);
        let twice = normalize_whitespace(&once);
        assert_eq!(once, twice, "Formatting is not idempotent");
    }
}
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

```rust
proptest! {
    #[test]
    fn fast_median_matches_naive_median(input in prop::collection::vec(any::<f64>(), 1..1000)) {
        let fast = fast_median(&input);
        let naive = naive_median(&input);
        assert!((fast - naive).abs() < 0.001);
    }
}
```

### Commutativity, Associativity, Distributivity

Mathematical properties that your operations should satisfy, if applicable.

### "No Crash" Property

The weakest but still useful property: the function does not panic for any valid input.

```rust
proptest! {
    #[test]
    fn parse_never_panics(input: String) {
        let _ = parse_config(&input);  // May return Err, but should never panic
    }
}
```

## Practical Considerations

### Test Speed

Property-based tests are slower than example-based tests because they run many iterations (default: 256 cases per test). For expensive operations, reduce the case count:

```rust
proptest! {
    #![proptest_config(ProptestConfig::with_cases(50))]

    #[test]
    fn expensive_operation_property(input: Vec<i32>) {
        // Only 50 cases instead of 256
        let result = expensive_operation(&input);
        assert!(result.is_valid());
    }
}
```

### Debugging Failures

When a property test fails, proptest prints the minimal failing input. Reproduce it with a targeted example-based test:

```rust
// proptest found: input = [0, 0]
#[test]
fn regression_dedup_with_duplicates() {
    let result = buggy_dedup(vec![0, 0]);
    assert_eq!(result, vec![0]);
}
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
