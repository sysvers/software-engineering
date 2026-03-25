# Security Testing

Security testing verifies that your application resists attack. It encompasses static analysis of source code, dynamic testing of running applications, fuzzing for unexpected inputs, and continuous scanning of dependencies. The goal is to find vulnerabilities before attackers do.

## Static Application Security Testing (SAST)

SAST analyzes source code for vulnerabilities without running the application. It catches issues early, integrates into CI, and runs fast. The trade-off is false positives and inability to detect runtime issues.

### cargo audit

Checks your `Cargo.lock` against the RustSec advisory database. This is the Rust equivalent of `npm audit` or Dependabot.

```bash
# Install
cargo install cargo-audit

# Check for known vulnerabilities
cargo audit

# Output example:
# Crate:     chrono
# Version:   0.4.19
# Warning:   unsound
# ID:        RUSTSEC-2020-0159
# URL:       https://rustsec.org/advisories/RUSTSEC-2020-0159
```

`cargo audit` should run on every pull request. A known vulnerability in a dependency is the easiest attack vector to exploit and the easiest to prevent. The Equifax breach (2017, 147 million records, $1.4 billion cost) happened because a known vulnerability in Apache Struts went unpatched for months. `cargo audit` in CI prevents this class of failure.

### cargo deny

Policy enforcement for dependencies. Goes beyond `cargo audit` to cover licenses, sources, and duplicate crates. Configuration lives in `deny.toml`.

```toml
# deny.toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause"]
confidence-threshold = 0.8

[bans]
multiple-versions = "warn"
deny = [
    # Deny specific crates known to be problematic
    { name = "openssl" },  # Prefer rustls
]

[sources]
unknown-registry = "deny"
unknown-git = "deny"
```

```bash
cargo install cargo-deny

# Check all policies
cargo deny check

# Check specific categories
cargo deny check advisories
cargo deny check licenses
cargo deny check bans
cargo deny check sources
```

### cargo clippy

Not strictly a security tool, but catches code patterns that can lead to security issues: integer overflow, unused `Result` values (ignoring errors), and unreachable patterns.

```bash
# Run with all warnings as errors
cargo clippy -- -D warnings

# Enable additional security-relevant lints
cargo clippy -- -W clippy::unwrap_used -W clippy::expect_used
```

Disallowing `unwrap()` and `expect()` in production code prevents panics that can cause denial of service or leak information in error messages.

## Dynamic Application Security Testing (DAST)

DAST tests the running application by sending requests and analyzing responses. It finds issues that static analysis cannot: misconfigurations, missing headers, improper error handling at runtime, and business logic flaws.

**Tools:**
- **OWASP ZAP** -- Free, open-source. Automated scanning, manual testing, and API fuzzing. Can be run headless in CI.
- **Burp Suite** -- Industry standard for professional penetration testing. Community edition is free; Pro edition adds automated scanning.
- **Nuclei** -- Template-based vulnerability scanner. Fast, extensible, and suitable for CI integration.

**Running OWASP ZAP in CI:**

```yaml
# GitHub Actions example
security-scan:
  runs-on: ubuntu-latest
  services:
    app:
      image: your-app:latest
      ports:
        - 8080:8080
  steps:
    - name: OWASP ZAP Baseline Scan
      uses: zaproxy/action-baseline@v0.9.0
      with:
        target: "http://app:8080"
        rules_file_name: "zap-rules.tsv"
        fail_action: "true"
```

DAST complements SAST. SAST finds code-level flaws; DAST finds deployment and configuration flaws. Use both.

## Fuzzing for Security

Fuzzing feeds random or mutated inputs to your code to discover crashes, panics, and unexpected behavior. In Rust, a panic in a server handler is a denial-of-service vulnerability. Fuzzing is especially valuable for parsers, deserializers, and any code that processes untrusted input.

### cargo-fuzz (libFuzzer)

```bash
cargo install cargo-fuzz

# Initialize fuzzing targets
cargo fuzz init

# Create a fuzz target
cargo fuzz add parse_input
```

```text
// fuzz/fuzz_targets/parse_input
// Fuzz target definition

FUZZ_TARGET(data : bytes):
    IF data IS VALID UTF-8 THEN
        s ← CONVERT_TO_STRING(data)
        // Fuzz the parser -- it should never panic
        CALL PARSE_INPUT(s)    // ignore result, just check it does not crash
    END IF
```

```bash
# Run the fuzzer (runs indefinitely until stopped or crash found)
cargo fuzz run parse_input

# Run with a time limit for CI
cargo fuzz run parse_input -- -max_total_time=300
```

### Property-based testing with proptest

Not traditional fuzzing, but serves a similar purpose: automatically generating inputs to find edge cases.

```text
// Property-based tests with automatically generated inputs

TEST parse_never_panics:
    FOR EACH input IN RANDOM_PRINTABLE_STRINGS DO
        // Property: PARSE_INPUT should never panic, regardless of input
        CALL PARSE_INPUT(input)    // ignore result, just verify no crash
    END FOR

TEST validated_email_roundtrips:
    FOR EACH email IN RANDOM_STRINGS MATCHING "[a-z]{1,10}@[a-z]{1,10}.[a-z]{2,4}" DO
        parsed ← EmailAddress.PARSE(email)
        ASSERT parsed.AS_STRING() = email
    END FOR
```

## Dependency Scanning

Dependencies are the largest attack surface for most applications. Most Rust projects pull in hundreds of transitive dependencies, each of which can introduce vulnerabilities.

### Automated dependency updates

**Dependabot** (GitHub) or **Renovate** (self-hosted) automatically open pull requests when new versions of your dependencies are available, including security patches.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "cargo"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

### Software Bill of Materials (SBOM)

An SBOM is a complete inventory of all components in your software, including transitive dependencies. After Log4Shell (2021), SBOMs became a critical part of supply chain security. Most affected organizations did not even know they depended on Log4j.

```bash
# Generate SBOM in CycloneDX format
cargo install cargo-cyclonedx
cargo cyclonedx --format json
```

### Secret scanning

Prevent credentials from being committed to version control.

```bash
# Install gitleaks
brew install gitleaks

# Scan repository history
gitleaks detect --source . --verbose

# Scan staged changes (use as pre-commit hook)
gitleaks protect --staged
```

GitHub's built-in secret scanning detects API keys, passwords, and tokens automatically for public repositories and with GitHub Advanced Security for private repositories.

## CI Integration

All security testing should run automatically. Manual security reviews do not scale and will be skipped under deadline pressure.

### Complete CI security pipeline

```yaml
# .github/workflows/security.yml
name: Security
on: [push, pull_request]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install cargo-audit
        run: cargo install cargo-audit
      - name: Audit dependencies
        run: cargo audit

  deny:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install cargo-deny
        run: cargo install cargo-deny
      - name: Check advisories
        run: cargo deny check advisories
      - name: Check licenses
        run: cargo deny check licenses

  clippy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Clippy with security lints
        run: cargo clippy -- -D warnings -W clippy::unwrap_used

  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Gitleaks
        uses: gitleaks/gitleaks-action@v2

  fuzz:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Install cargo-fuzz
        run: cargo install cargo-fuzz
      - name: Fuzz parsers (5 min)
        run: cargo fuzz run parse_input -- -max_total_time=300
```

### What to run when

| Check | On every PR | On merge to main | Weekly |
|-------|-------------|-------------------|--------|
| `cargo audit` | Yes | Yes | Yes |
| `cargo deny check` | Yes | Yes | Yes |
| `cargo clippy` | Yes | Yes | -- |
| Secret scanning | Yes | Yes | Yes (full history) |
| Fuzzing (short) | Optional | Yes | -- |
| Fuzzing (long) | -- | -- | Yes |
| DAST scan | -- | Yes (staging) | Yes |
| Full penetration test | -- | -- | Quarterly |

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **SAST** | Catches bugs early, fast, integrates into CI | False positives, cannot detect runtime issues |
| **DAST** | Finds runtime misconfigurations, tests real behavior | Slower, requires running application, may miss code-level flaws |
| **Fuzzing** | Finds edge cases humans miss, especially in parsers | Requires setup, can be slow, non-deterministic |
| **Dependency scanning** | Catches known vulnerabilities automatically | Only finds *known* vulnerabilities, not zero-days |
| **Secret scanning** | Prevents credential leaks | False positives on test data, cannot find secrets already rotated |

## Key Takeaways

1. Run `cargo audit` and `cargo deny` on every pull request. This is non-negotiable for any project with dependencies.
2. SAST and DAST are complementary, not alternatives. Use both.
3. Fuzz all code that processes untrusted input: parsers, deserializers, validators.
4. Automate everything. Security checks that require manual action will be skipped under pressure.
5. Generate and maintain an SBOM. When the next Log4Shell happens, you need to know within minutes whether you are affected.
6. Prevent secrets from entering version control. Use pre-commit hooks with gitleaks and enable GitHub's built-in secret scanning.
