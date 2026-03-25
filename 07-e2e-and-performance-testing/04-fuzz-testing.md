# Fuzz Testing

## What Is Fuzz Testing?

Fuzz testing (fuzzing) feeds random, unexpected, or malformed input to your program to find crashes, panics, and security vulnerabilities that you would never think to test for manually.

Where unit tests verify **expected** behavior with **known** inputs, fuzz testing explores the vast space of **unexpected** inputs that real-world attackers and edge cases produce.

### Why It Matters

- Finds edge cases that human testers miss — inputs no developer would think to try
- Discovers buffer overflows, parsing errors, integer overflows, and panic-inducing inputs
- Especially valuable for parsers, deserializers, network protocols, and any code handling untrusted input
- Has found thousands of real security vulnerabilities in production software

---

## Coverage-Guided Fuzzing

Modern fuzzers don't just throw random bytes. They use **coverage-guided fuzzing** — tracking which code paths each input exercises, then generating new inputs that explore unexplored paths.

### How It Works

```
1. Start with a corpus of seed inputs (valid examples)
2. Mutate an input (flip bits, insert bytes, splice inputs together)
3. Run the target function with the mutated input
4. Track code coverage — did this input reach new code paths?
5. If YES → save the input to the corpus (it's interesting)
6. If NO → discard it
7. Repeat millions of times
```

This feedback loop means the fuzzer progressively discovers deeper and more complex code paths. It doesn't just test the surface — it systematically explores the entire reachable state space.

### The Engine: libFuzzer

[libFuzzer](https://llvm.org/docs/LibFuzzer.html) is LLVM's coverage-guided fuzzing engine. It instruments your code at compile time to track coverage, then uses that feedback to guide input generation.

In Rust, `cargo-fuzz` wraps libFuzzer with a convenient interface.

---

## Fuzz Testing in Rust with cargo-fuzz

### Setup

```bash
# Install cargo-fuzz (requires nightly Rust)
cargo install cargo-fuzz

# Initialize fuzzing in your project
cargo fuzz init

# Add a fuzz target
cargo fuzz add parse_config
```

This creates the directory structure:

```
fuzz/
  Cargo.toml
  fuzz_targets/
    parse_config.rs
```

### Writing a Fuzz Target

A fuzz target is a function that takes arbitrary bytes and feeds them to the code under test:

```text
// fuzz/fuzz_targets/parse_config
FUZZ TARGET (data: bytes)
    // Convert raw bytes to a string (skip invalid UTF-8)
    IF data IS valid UTF-8
        input <- data AS string
        // The fuzzer will try millions of inputs
        // If PARSE_CONFIG panics on ANY input, the fuzzer reports it
        CALL PARSE_CONFIG(input)
```

### Running the Fuzzer

```bash
# Run the fuzzer (runs indefinitely until stopped or crash found)
cargo +nightly fuzz run parse_config

# Run with a time limit
cargo +nightly fuzz run parse_config -- -max_total_time=300  # 5 minutes

# Run with a corpus directory (seed inputs)
cargo +nightly fuzz run parse_config corpus/parse_config/
```

### Using Structured Inputs with Arbitrary

For more targeted fuzzing, use the `arbitrary` crate to generate structured inputs instead of raw bytes:

```text
STRUCTURE FuzzInput (auto-generated from random data)
    name: string
    age: unsigned integer
    tags: list of string
    nested: optional FuzzInput

FUZZ TARGET (input: FuzzInput)
    // The fuzzer generates valid FuzzInput structs
    // with random but well-typed data
    CALL PROCESS_USER_INPUT(input.name, input.age, input.tags)
```

This produces more meaningful inputs than raw bytes, reaching deeper into your application logic.

---

## Practical Examples

### Fuzzing a JSON Parser

```text
FUZZ TARGET (data: bytes)
    IF data IS valid UTF-8
        input <- data AS string
        // Should never panic, regardless of input
        CALL JSON_PARSE(input)
```

This tests that your JSON parser handles every possible input without panicking. Real JSON parsers have been found to panic on deeply nested structures, specific Unicode sequences, and extremely long number literals.

### Fuzzing a Network Protocol Parser

```text
FUZZ TARGET (data: bytes)
    // Network protocols receive untrusted bytes from the network
    // They must handle ANY input safely
    CALL PARSE_MESSAGE(data)
```

### Fuzzing with Round-Trip Invariants

A powerful pattern: verify that encoding then decoding produces the original value.

```text
FUZZ TARGET (original: list of bytes)
    encoded <- ENCODE(original)
    decoded <- DECODE(encoded)  // must succeed for valid encoded data
    ASSERT original = decoded, "round-trip must preserve data"
```

If this assertion fails on any input, you have found a real bug in your codec.

### Fuzzing Comparison: Differential Testing

Compare two implementations of the same function to find discrepancies:

```text
FUZZ TARGET (data: bytes)
    IF data IS valid UTF-8
        input <- data AS string
        result_v1 <- PARSER_V1.PARSE(input)
        result_v2 <- PARSER_V2.PARSE(input)
        ASSERT result_v1 = result_v2, "v1 and v2 disagree on input: " + input
```

This finds inputs where your new implementation differs from the old one — invaluable during rewrites.

---

## Security Bugs Found by Fuzzing

Fuzzing has found critical vulnerabilities in real-world software:

### Heartbleed-Class Bugs

Buffer over-reads where the program reads past the end of allocated memory. Fuzzers find these by generating inputs with length fields that don't match actual data lengths:

```
Input: "LENGTH=1000" but only 5 bytes of data follow
Result: Program reads 995 bytes of adjacent memory (information leak)
```

### Integer Overflows

```text
// Vulnerable code
FUNCTION ALLOCATE_BUFFER(width: unsigned integer, height: unsigned integer) -> list of bytes
    size <- width * height * 4  // Can overflow!
    RETURN NEW list of zeroed bytes WITH length size
```

A fuzzer might generate `width = 65536, height = 65536`, causing `width * height * 4` to overflow, allocating a tiny buffer that subsequent writes overflow.

### Panic in Production

In Rust, panics don't cause memory corruption, but they do crash the program. For servers, a panic in a request handler crashes the worker or the entire process:

```text
// Found by fuzzing: panic on empty input
FUNCTION PARSE_HEADER(data: bytes) -> Header
    version <- data[0]                          // panics if data is empty
    length <- BYTES_TO_U32(data[1..5])          // panics if < 5 bytes
    // ...
```

A fuzzer finds this instantly. A unit test might never test with empty input because the developer assumed "the network layer validates minimum length."

### OSS-Fuzz Successes

Google's [OSS-Fuzz](https://github.com/google/oss-fuzz) project continuously fuzzes open-source software. As of 2025, it has found over 10,000 bugs across 1,000+ projects, including:

- Memory safety bugs in OpenSSL, SQLite, and curl
- Logic errors in compression libraries (zlib, brotli)
- Parsing crashes in image libraries (libpng, libjpeg-turbo)
- Protocol handling bugs in networking libraries

---

## Managing a Fuzz Corpus

The **corpus** is the set of inputs the fuzzer has found interesting (they reached new code paths). Managing it well improves fuzzing effectiveness.

### Seed Corpus

Start with real-world examples of valid input:

```bash
# Create seed corpus directory
mkdir -p fuzz/corpus/parse_config/

# Add real config files as seeds
cp examples/valid_config.toml fuzz/corpus/parse_config/seed1
cp examples/minimal_config.toml fuzz/corpus/parse_config/seed2
```

Seeds give the fuzzer a head start — it mutates them to find crashes faster than starting from random bytes.

### Minimizing Crash Inputs

When the fuzzer finds a crash, the triggering input is often large and contains irrelevant bytes. Minimize it:

```bash
# Minimize a crash-triggering input
cargo +nightly fuzz tmin parse_config fuzz/artifacts/parse_config/crash-abc123
```

This produces the smallest input that still triggers the crash, making debugging easier.

### Reproducing Crashes

```bash
# Reproduce a specific crash
cargo +nightly fuzz run parse_config fuzz/artifacts/parse_config/crash-abc123
```

Then write a regression test so the bug stays fixed:

```text
TEST regression_crash_empty_header
    // Found by fuzzing: parse_header panicked on empty input
    result <- PARSE_HEADER(empty bytes)
    ASSERT result IS error  // Must return Err, not panic
```

---

## Common Pitfalls

### Fuzzing Too Broadly

Fuzzing raw bytes against a high-level API is inefficient. The fuzzer spends most of its time generating invalid inputs rejected immediately. Use `Arbitrary` for structured fuzzing to get past input validation and into the interesting logic.

### Ignoring Timeouts

Some inputs cause the program to run for minutes (e.g., pathological regex patterns, deeply nested structures). Set timeouts:

```bash
cargo +nightly fuzz run parse_config -- -timeout=5  # 5-second timeout per input
```

### Not Running Long Enough

A 30-second fuzzing run might find obvious crashes. Deeper bugs require hours or days of fuzzing. Run fuzzers overnight or as a continuous background process in CI.

### Not Saving the Corpus

The corpus represents weeks of exploration. Save it to version control or artifact storage. Losing the corpus means starting from scratch.

---

## When to Use Fuzz Testing

**Use for:**
- Parsers and deserializers (JSON, XML, TOML, custom formats)
- Network protocol handlers (anything receiving bytes from the network)
- Codecs (compression, encryption, encoding)
- Any code processing untrusted or user-supplied input
- Security-sensitive code

**Avoid for:**
- Business logic (use unit tests with known inputs)
- UI code (fuzzing clicks doesn't find meaningful bugs)
- Code that only processes trusted, internally-generated data

---

## Key Takeaways

1. Fuzz testing finds bugs you would never think to test for. It systematically explores inputs humans can't imagine.
2. Coverage-guided fuzzing (libFuzzer, cargo-fuzz) is far more effective than random input generation.
3. Use `Arbitrary` for structured fuzzing to bypass input validation and test deeper logic.
4. Always write regression tests for crashes found by fuzzing — they are real bugs that will recur.
5. Run fuzzers for hours or days, not seconds. Deep bugs require sustained exploration.
6. Fuzzing is particularly high-value for any code that handles untrusted input — parsers, protocols, and codecs.
