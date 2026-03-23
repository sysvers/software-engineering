# Rust-Specific Patterns

These patterns either originate in Rust or take a form unique to the language due to its ownership model, type system, and zero-cost abstractions. They exploit Rust's compile-time guarantees to catch entire categories of bugs that other languages can only detect at runtime.

---

## Newtype Pattern

### What It Is

Wrap a primitive type in a single-field struct to give it a distinct type identity. The wrapper has zero runtime cost (same memory layout as the inner type) but prevents accidental misuse at compile time.

### The Problem

```rust
// Without newtype -- dangerously easy to mix up
fn transfer(from: u64, to: u64, amount: u64) {
    println!("Transferring {amount} from {from} to {to}");
}

// This compiles and runs. It is completely wrong.
let user_id = 42;
let account_id = 100;
let cents = 5000;

transfer(cents, user_id, account_id);
// "Transferring 100 from 5000 to 42" -- silent corruption
```

### The Solution

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct UserId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct AccountId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
struct Cents(u64);

fn transfer(from: AccountId, to: AccountId, amount: Cents) {
    println!("Transferring {:?} from {:?} to {:?}", amount, from, to);
}

let user = UserId(42);
let sender = AccountId(100);
let receiver = AccountId(200);
let amount = Cents(5000);

// transfer(amount, user, sender);  // Compiler error!
// transfer(sender, receiver, user); // Compiler error!
transfer(sender, receiver, amount);  // Correct -- compiles
```

### Adding Behavior to Newtypes

```rust
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Temperature(f64);

impl Temperature {
    fn from_celsius(c: f64) -> Self {
        Self(c)
    }

    fn from_fahrenheit(f: f64) -> Self {
        Self((f - 32.0) * 5.0 / 9.0)
    }

    fn as_celsius(&self) -> f64 {
        self.0
    }

    fn as_fahrenheit(&self) -> f64 {
        self.0 * 9.0 / 5.0 + 32.0
    }

    fn is_freezing(&self) -> bool {
        self.0 <= 0.0
    }
}

// Cannot accidentally add Celsius to Fahrenheit
let boiling = Temperature::from_celsius(100.0);
let body_temp = Temperature::from_fahrenheit(98.6);

// Both are Temperature -- comparison is meaningful because both store Celsius internally
assert!(boiling > body_temp);
```

### Implementing Traits for Foreign Types

The newtype pattern also solves Rust's orphan rule: you cannot implement a foreign trait for a foreign type, but you can wrap the foreign type in a newtype:

```rust
// Cannot do: impl fmt::Display for Vec<String> -- both are foreign
// But can do:

struct CommaSeparated(Vec<String>);

impl fmt::Display for CommaSeparated {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", self.0.join(", "))
    }
}

let tags = CommaSeparated(vec!["rust".into(), "patterns".into(), "newtype".into()]);
println!("{tags}"); // "rust, patterns, newtype"
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

```rust
use std::marker::PhantomData;

// State markers -- zero-sized, exist only in the type system
struct Draft;
struct UnderReview;
struct Approved;
struct Published;

// The document carries its state as a type parameter
struct Document<State> {
    title: String,
    content: String,
    author: String,
    _state: PhantomData<State>,
}

// Methods available in ALL states
impl<S> Document<S> {
    pub fn title(&self) -> &str { &self.title }
    pub fn content(&self) -> &str { &self.content }
    pub fn author(&self) -> &str { &self.author }
}

// Methods available ONLY in Draft state
impl Document<Draft> {
    pub fn new(title: &str, author: &str) -> Self {
        Self {
            title: title.to_string(),
            content: String::new(),
            author: author.to_string(),
            _state: PhantomData,
        }
    }

    pub fn edit(&mut self, content: &str) {
        self.content = content.to_string();
    }

    pub fn submit_for_review(self) -> Document<UnderReview> {
        Document {
            title: self.title,
            content: self.content,
            author: self.author,
            _state: PhantomData,
        }
    }
}

// Methods available ONLY in UnderReview state
impl Document<UnderReview> {
    pub fn approve(self) -> Document<Approved> {
        Document {
            title: self.title,
            content: self.content,
            author: self.author,
            _state: PhantomData,
        }
    }

    pub fn reject(self, feedback: &str) -> Document<Draft> {
        println!("Rejected with feedback: {feedback}");
        Document {
            title: self.title,
            content: self.content,
            author: self.author,
            _state: PhantomData,
        }
    }
}

// Methods available ONLY in Approved state
impl Document<Approved> {
    pub fn publish(self) -> Document<Published> {
        println!("Publishing: {}", self.title);
        Document {
            title: self.title,
            content: self.content,
            author: self.author,
            _state: PhantomData,
        }
    }
}

// Methods available ONLY in Published state
impl Document<Published> {
    pub fn url(&self) -> String {
        format!("/articles/{}", self.title.to_lowercase().replace(' ', "-"))
    }
}

// Usage -- the compiler enforces the workflow
let doc = Document::<Draft>::new("Rust Typestate", "Alice");

// doc.publish();          // Compile error! Draft cannot be published
// doc.approve();          // Compile error! Draft cannot be approved

let mut doc = doc;
doc.edit("Typestate is a pattern where...");
let doc = doc.submit_for_review();

// doc.edit("changes");    // Compile error! Cannot edit while under review

let doc = doc.approve();
let doc = doc.publish();

println!("Published at: {}", doc.url());

// doc.edit("more changes"); // Compile error! Cannot edit published document
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

```rust
let builder = ThingBuilder::new();
let a = builder.name("A").build(); // builder is moved
// let b = builder.name("B").build(); // Compile error! builder was consumed
```

### Rust Implementation with Typestate

```rust
use std::marker::PhantomData;

// Markers for required field status
struct Missing;
struct Set;

struct ConnectionBuilder<HasHost, HasPort> {
    host: Option<String>,
    port: Option<u16>,
    timeout_ms: u64,
    tls: bool,
    max_retries: u32,
    _has_host: PhantomData<HasHost>,
    _has_port: PhantomData<HasPort>,
}

impl ConnectionBuilder<Missing, Missing> {
    pub fn new() -> Self {
        Self {
            host: None,
            port: None,
            timeout_ms: 30_000,
            tls: true,
            max_retries: 3,
            _has_host: PhantomData,
            _has_port: PhantomData,
        }
    }
}

impl<P> ConnectionBuilder<Missing, P> {
    /// Set the host (required). Transitions HasHost from Missing to Set.
    pub fn host(self, host: &str) -> ConnectionBuilder<Set, P> {
        ConnectionBuilder {
            host: Some(host.to_string()),
            port: self.port,
            timeout_ms: self.timeout_ms,
            tls: self.tls,
            max_retries: self.max_retries,
            _has_host: PhantomData,
            _has_port: PhantomData,
        }
    }
}

impl<H> ConnectionBuilder<H, Missing> {
    /// Set the port (required). Transitions HasPort from Missing to Set.
    pub fn port(self, port: u16) -> ConnectionBuilder<H, Set> {
        ConnectionBuilder {
            host: self.host,
            port: Some(port),
            timeout_ms: self.timeout_ms,
            tls: self.tls,
            max_retries: self.max_retries,
            _has_host: PhantomData,
            _has_port: PhantomData,
        }
    }
}

// Optional fields -- available regardless of state
impl<H, P> ConnectionBuilder<H, P> {
    pub fn timeout(mut self, ms: u64) -> Self {
        self.timeout_ms = ms;
        self
    }

    pub fn tls(mut self, enabled: bool) -> Self {
        self.tls = enabled;
        self
    }

    pub fn max_retries(mut self, n: u32) -> Self {
        self.max_retries = n;
        self
    }
}

// build() only available when BOTH required fields are set
impl ConnectionBuilder<Set, Set> {
    pub fn build(self) -> Connection {
        Connection {
            host: self.host.unwrap(),
            port: self.port.unwrap(),
            timeout_ms: self.timeout_ms,
            tls: self.tls,
            max_retries: self.max_retries,
        }
    }
}

struct Connection {
    host: String,
    port: u16,
    timeout_ms: u64,
    tls: bool,
    max_retries: u32,
}

// Usage
let conn = ConnectionBuilder::new()
    .host("db.example.com")
    .port(5432)
    .timeout(10_000)
    .tls(true)
    .build(); // Compiles -- both host and port are set

// This does NOT compile -- port is missing:
// let conn = ConnectionBuilder::new()
//     .host("db.example.com")
//     .build(); // Error: build() not available on ConnectionBuilder<Set, Missing>
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
