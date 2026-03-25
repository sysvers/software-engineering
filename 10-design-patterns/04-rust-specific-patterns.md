# Rust-Specific Patterns

These patterns either originate in Rust or take a form unique to the language due to its ownership model, type system, and zero-cost abstractions. They exploit Rust's compile-time guarantees to catch entire categories of bugs that other languages can only detect at runtime.

---

## Newtype Pattern

### What It Is

Wrap a primitive type in a single-field struct to give it a distinct type identity. The wrapper has zero runtime cost (same memory layout as the inner type) but prevents accidental misuse at compile time.

### The Problem

```text
// Without newtype -- dangerously easy to mix up
FUNCTION TRANSFER(from: Integer, to: Integer, amount: Integer)
    PRINT "Transferring " + amount + " from " + from + " to " + to

// This compiles and runs. It is completely wrong.
user_id ← 42
account_id ← 100
cents ← 5000

TRANSFER(cents, user_id, account_id)
// "Transferring 100 from 5000 to 42" -- silent corruption
```

### The Solution

```text
// Define distinct types wrapping Integer
TYPE UserId = WRAPPER(Integer)
TYPE AccountId = WRAPPER(Integer)
TYPE Cents = WRAPPER(Integer)

FUNCTION TRANSFER(from: AccountId, to: AccountId, amount: Cents)
    PRINT "Transferring " + amount + " from " + from + " to " + to

user ← UserId(42)
sender ← AccountId(100)
receiver ← AccountId(200)
amount ← Cents(5000)

// TRANSFER(amount, user, sender)    // Type error!
// TRANSFER(sender, receiver, user)  // Type error!
TRANSFER(sender, receiver, amount)   // Correct -- types match
```

### Adding Behavior to Newtypes

```text
TYPE Temperature = WRAPPER(Float)

FUNCTION FROM_CELSIUS(c: Float) → Temperature
    RETURN Temperature(c)

FUNCTION FROM_FAHRENHEIT(f: Float) → Temperature
    RETURN Temperature((f - 32.0) * 5.0 / 9.0)

FUNCTION AS_CELSIUS(t: Temperature) → Float
    RETURN t.value

FUNCTION AS_FAHRENHEIT(t: Temperature) → Float
    RETURN t.value * 9.0 / 5.0 + 32.0

FUNCTION IS_FREEZING(t: Temperature) → Boolean
    RETURN t.value ≤ 0.0

// Cannot accidentally add Celsius to Fahrenheit
boiling ← FROM_CELSIUS(100.0)
body_temp ← FROM_FAHRENHEIT(98.6)

// Both are Temperature -- comparison is meaningful because both store Celsius internally
ASSERT boiling > body_temp
```

### Implementing Traits for Foreign Types

The newtype pattern also solves Rust's orphan rule: you cannot implement a foreign trait for a foreign type, but you can wrap the foreign type in a newtype:

```text
// Cannot implement Display for List<String> directly -- both are foreign
// But can wrap it:

TYPE CommaSeparated = WRAPPER(List<String>)

FUNCTION TO_STRING(cs: CommaSeparated) → String
    RETURN JOIN(cs.items, ", ")

tags ← CommaSeparated(["rust", "patterns", "newtype"])
PRINT TO_STRING(tags)  // "rust, patterns, newtype"
```

### Real-World Crates That Use It

- **uuid::Uuid** -- A newtype around `[u8; 16]` that provides UUID-specific formatting, parsing, and version generation.
- **std::num::NonZeroU64** -- A newtype that guarantees the value is never zero, enabling `Option<NonZeroU64>` to have the same size as `u64` (niche optimization).
- **url::Url** -- A newtype around `String` that guarantees the string is a valid URL. Construction validates; all methods can assume validity.
- **chrono::Duration** vs **std::time::Duration** -- Different newtypes over time quantities with different semantics (signed vs. unsigned).
- **axum::extract::Path, Query, Json** -- Newtypes that tell the framework how to extract data from an HTTP request.

### Pitfalls

1. **Boilerplate** -- Each newtype needs `derive` macros and potentially `Deref`, `From`, `Display`, and arithmetic trait implementations. Use `derive_more` or `nutype` crates to reduce boilerplate.
2. **Over-wrapping** -- Not every `u64` needs a newtype. Wrap types that cross API boundaries or where mix-ups have occurred (or are likely).
3. **Deref abuse** -- Implementing `Deref<Target = InnerType>` makes the newtype transparent, which can defeat the purpose. Use it sparingly.
4. **Serialization** -- Newtypes need `serde::Serialize`/`Deserialize` implementations. The `#[serde(transparent)]` attribute handles this cleanly.

### When to Use / When to Avoid

**Use when:**
- Function parameters have the same primitive type but different meanings (IDs, amounts, coordinates)
- You need to implement a foreign trait for a foreign type
- You want to enforce invariants at the type level (NonZero, ValidEmail, Positive)

**Avoid when:**
- The type is used in one place and mix-ups are impossible
- The boilerplate cost outweighs the safety benefit (internal one-off calculations)

---

## Typestate Pattern

### What It Is

Encode the state of an object in its type, so that the compiler prevents invalid state transitions at compile time. Unlike the enum-based state pattern (runtime checks), typestate makes illegal states unrepresentable -- the code literally cannot compile if you try to perform an invalid transition.

### The Mechanism

The key ingredients are:
1. **Zero-sized type (ZST) marker types** for each state
2. **PhantomData** to "use" the type parameter without storing data
3. **Methods only on specific state types** via `impl<State> Struct<State>` blocks

### Rust Implementation

```text
// State markers -- zero-sized, exist only in the type system
STATE TYPE Draft
STATE TYPE UnderReview
STATE TYPE Approved
STATE TYPE Published

// The document carries its state as a type parameter
RECORD Document<State>
    title: String
    content: String
    author: String

// Methods available in ALL states
FUNCTION TITLE(doc: Document<Any>) → String
    RETURN doc.title

FUNCTION CONTENT(doc: Document<Any>) → String
    RETURN doc.content

FUNCTION AUTHOR(doc: Document<Any>) → String
    RETURN doc.author

// Methods available ONLY in Draft state
FUNCTION NEW_DOCUMENT(title: String, author: String) → Document<Draft>
    RETURN Document<Draft> { title ← title, content ← "", author ← author }

FUNCTION EDIT(doc: Document<Draft>, content: String) → Document<Draft>
    doc.content ← content
    RETURN doc

FUNCTION SUBMIT_FOR_REVIEW(doc: Document<Draft>) → Document<UnderReview>
    // Consumes the Draft document, returns UnderReview document
    RETURN Document<UnderReview> { title ← doc.title, content ← doc.content, author ← doc.author }

// Methods available ONLY in UnderReview state
FUNCTION APPROVE(doc: Document<UnderReview>) → Document<Approved>
    RETURN Document<Approved> { title ← doc.title, content ← doc.content, author ← doc.author }

FUNCTION REJECT(doc: Document<UnderReview>, feedback: String) → Document<Draft>
    PRINT "Rejected with feedback: " + feedback
    RETURN Document<Draft> { title ← doc.title, content ← doc.content, author ← doc.author }

// Methods available ONLY in Approved state
FUNCTION PUBLISH(doc: Document<Approved>) → Document<Published>
    PRINT "Publishing: " + doc.title
    RETURN Document<Published> { title ← doc.title, content ← doc.content, author ← doc.author }

// Methods available ONLY in Published state
FUNCTION URL(doc: Document<Published>) → String
    RETURN "/articles/" + LOWERCASE(REPLACE(doc.title, " ", "-"))

// Usage -- the compiler enforces the workflow
doc ← NEW_DOCUMENT("Rust Typestate", "Alice")

// PUBLISH(doc)              // Compile error! Draft cannot be published
// APPROVE(doc)              // Compile error! Draft cannot be approved

doc ← EDIT(doc, "Typestate is a pattern where...")
doc ← SUBMIT_FOR_REVIEW(doc)

// EDIT(doc, "changes")      // Compile error! Cannot edit while under review

doc ← APPROVE(doc)
doc ← PUBLISH(doc)

PRINT "Published at: " + URL(doc)

// EDIT(doc, "more changes") // Compile error! Cannot edit published document
```

### Why This Is Unique to Rust

Most languages cannot express typestate because:
1. **Move semantics** -- When `submit_for_review(self)` consumes `self`, the old `Document<Draft>` ceases to exist. In garbage-collected languages, nothing prevents holding a reference to the old state.
2. **Zero-cost abstractions** -- The `PhantomData` and ZST markers are completely erased at compile time. The runtime representation of `Document<Draft>` and `Document<Published>` is identical.
3. **No null/nil** -- You cannot have an "uninitialized" document. Every document is in exactly one state.

### Real-World Crates That Use It

- **hyper::Request / http::request::Builder** -- The HTTP request builder uses typestate to ensure you cannot send a request without a method and URI. `Request::builder().method("GET").uri("/").body(())` -- each step returns a different type.
- **diesel query builder** -- Diesel uses typestate to ensure you cannot call `.execute()` on a SELECT query or `.load()` on an INSERT query. The query type carries its "kind" in the type parameter.
- **rocket::Request** -- Rocket uses typestate-like patterns to track which parts of a request have been parsed.
- **rusoto / aws-sdk-rust** -- AWS SDK builders enforce required fields through the type system. You cannot call `.send()` until all required fields are set.
- **typestate crate** -- The `typestate` proc-macro crate automates typestate pattern generation from annotated enums.

### Pitfalls

1. **Type complexity** -- `Document<Draft>`, `Document<UnderReview>`, `Document<Approved>` are different types. Functions that accept "any document" need generics: `fn title<S>(doc: &Document<S>) -> &str`.
2. **Cannot store mixed states in a collection** -- `Vec<Document<???>>` does not work because all elements must have the same type. Use an enum wrapper if you need mixed collections.
3. **Code duplication** -- State transitions that carry data involve repeating field copies. Consider a macro or a helper function to reduce duplication.
4. **Learning curve** -- Typestate is unfamiliar to developers from other languages. Document the state machine clearly.

### When to Use / When to Avoid

**Use when:**
- Invalid state transitions are bugs that must be prevented at compile time
- The state machine is central to correctness (protocol implementations, workflow engines)
- States have genuinely different APIs (not just different behavior on the same methods)

**Avoid when:**
- The state is determined at runtime (user input, database values) -- use an enum instead
- There are many states and transitions (10+ states) -- the type complexity becomes unwieldy
- The states share 95% of their API -- the overhead of separate impl blocks is not worth it

---

## Builder with Consume (Consuming Builder)

### What It Is

A refinement of the builder pattern where each method takes `self` by value (consuming it) rather than by mutable reference. This ensures each builder is used exactly once and enables compile-time enforcement of required fields through typestate.

### Why It Matters in Rust

In languages with garbage collection, a builder taken by reference can be reused accidentally:

```java
// Java -- this is a subtle bug
Builder b = new Builder();
Thing a = b.name("A").build();
Thing oops = b.name("B").build(); // b still has name="A" state from before? or "B"? unclear
```

In Rust, the consuming builder makes reuse impossible:

```text
builder ← THING_BUILDER_NEW()
a ← BUILD(SET_NAME(builder, "A"))    // builder is consumed
// b ← BUILD(SET_NAME(builder, "B")) // Compile error! builder was already consumed
```

### Rust Implementation with Typestate

```text
// Markers for required field status
STATE TYPE Missing
STATE TYPE Set

RECORD ConnectionBuilder<HasHost, HasPort>
    host: Optional<String>
    port: Optional<Integer>
    timeout_ms: Integer
    tls: Boolean
    max_retries: Integer

FUNCTION NEW_CONNECTION_BUILDER() → ConnectionBuilder<Missing, Missing>
    RETURN ConnectionBuilder<Missing, Missing> {
        host ← NONE,
        port ← NONE,
        timeout_ms ← 30000,
        tls ← TRUE,
        max_retries ← 3
    }

// Set the host (required). Transitions HasHost from Missing to Set.
FUNCTION SET_HOST(builder: ConnectionBuilder<Missing, P>, host: String) → ConnectionBuilder<Set, P>
    RETURN ConnectionBuilder<Set, P> {
        host ← host,
        port ← builder.port,
        timeout_ms ← builder.timeout_ms,
        tls ← builder.tls,
        max_retries ← builder.max_retries
    }

// Set the port (required). Transitions HasPort from Missing to Set.
FUNCTION SET_PORT(builder: ConnectionBuilder<H, Missing>, port: Integer) → ConnectionBuilder<H, Set>
    RETURN ConnectionBuilder<H, Set> {
        host ← builder.host,
        port ← port,
        timeout_ms ← builder.timeout_ms,
        tls ← builder.tls,
        max_retries ← builder.max_retries
    }

// Optional fields -- available regardless of state
FUNCTION SET_TIMEOUT(builder: ConnectionBuilder<H, P>, ms: Integer) → ConnectionBuilder<H, P>
    builder.timeout_ms ← ms
    RETURN builder

FUNCTION SET_TLS(builder: ConnectionBuilder<H, P>, enabled: Boolean) → ConnectionBuilder<H, P>
    builder.tls ← enabled
    RETURN builder

FUNCTION SET_MAX_RETRIES(builder: ConnectionBuilder<H, P>, n: Integer) → ConnectionBuilder<H, P>
    builder.max_retries ← n
    RETURN builder

// BUILD only available when BOTH required fields are set
FUNCTION BUILD(builder: ConnectionBuilder<Set, Set>) → Connection
    RETURN Connection {
        host ← builder.host,
        port ← builder.port,
        timeout_ms ← builder.timeout_ms,
        tls ← builder.tls,
        max_retries ← builder.max_retries
    }

RECORD Connection
    host: String
    port: Integer
    timeout_ms: Integer
    tls: Boolean
    max_retries: Integer

// Usage
conn ← BUILD(
    SET_TLS(
        SET_TIMEOUT(
            SET_PORT(
                SET_HOST(NEW_CONNECTION_BUILDER(), "db.example.com"),
                5432),
            10000),
        TRUE))
// Compiles -- both host and port are set

// This does NOT compile -- port is missing:
// conn ← BUILD(SET_HOST(NEW_CONNECTION_BUILDER(), "db.example.com"))
// Error: BUILD not available on ConnectionBuilder<Set, Missing>
```

### Real-World Crates That Use It

- **typed-builder** -- A derive macro that generates consuming builders with compile-time required field checking. Used by many Rust libraries for ergonomic, safe struct construction.
- **derive_builder** -- Generates builders, with an option for consuming (owned) style.
- **bon** -- A newer builder crate that uses typestate under the hood for required vs optional fields.
- **tonic (gRPC)** -- Generated gRPC client builders consume self, ensuring the request is fully configured before sending.

### Pitfalls

1. **Verbose type signatures** -- `ConnectionBuilder<Set, Missing>` appears in error messages, which can confuse users. Good error messages and documentation help.
2. **Exponential impl blocks** -- With N required fields, you need impl blocks for each combination. Macros or proc-macros (like `typed-builder`) automate this.
3. **Cannot reuse builder** -- This is by design, but it means you cannot build two similar objects from one builder. Implement `Clone` on the builder if reuse is needed.
4. **IDE support** -- Some IDEs struggle with typestate builders, showing incomplete autocompletion. This is improving but still a friction point.

### When to Use / When to Avoid

**Use when:**
- Missing required fields should be a compile-time error, not a runtime panic
- The builder is part of a public API where misuse must be prevented
- You want to guarantee each builder produces exactly one object

**Avoid when:**
- The builder is internal and used in 2-3 places -- runtime validation is fine
- You need to build multiple objects from one base configuration (clone-based builder is better)
- The number of required fields is very large (5+) -- the type parameter list becomes unwieldy; consider grouping required fields into a config struct passed to `new()`
