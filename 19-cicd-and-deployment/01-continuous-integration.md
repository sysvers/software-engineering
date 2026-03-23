# Continuous Integration

## What Is CI?

CI is the practice of merging all developers' work into a shared mainline frequently (at least daily), with each merge validated by automated builds and tests.

The goal is simple: catch problems early, when they are cheap to fix. A bug caught in CI costs minutes; the same bug caught in production costs hours (or worse).

```
Push to branch → [Build] → [Lint] → [Unit Tests] → [Integration Tests] → [Security Scan] → Ready for review
```

## What CI Should Validate

Every CI pipeline for a Rust project should check at least these five things:

| Check | Tool | Why |
|-------|------|-----|
| **Compilation** | `cargo build` | Does the code compile at all? In Rust the type system catches entire categories of bugs here. |
| **Formatting** | `cargo fmt --check` | Consistent style eliminates cosmetic diffs in reviews. |
| **Linting** | `cargo clippy -- -D warnings` | Catches common mistakes, performance issues, and unidiomatic patterns. |
| **Tests** | `cargo nextest run` | Unit and integration tests pass. `nextest` is faster than `cargo test` for large suites. |
| **Security audit** | `cargo audit` | Flags dependencies with known vulnerabilities (CVEs). |

Optional but valuable additions:

- **Code coverage** (`cargo llvm-cov`) — track whether new code has tests.
- **Dependency review** — detect license violations or supply-chain risks.
- **Documentation build** (`cargo doc --no-deps`) — catch broken doc links before merge.
- **Benchmarks** — detect performance regressions with `criterion` or `divan`.

## GitHub Actions CI for Rust

A production-ready workflow that covers all the checks above:

```yaml
name: CI
on: [push, pull_request]

env:
  CARGO_TERM_COLOR: always
  # Pin the nightly date for reproducibility.
  RUSTFLAGS: "-D warnings"

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt

      # Cache dependencies to avoid rebuilding every run.
      - uses: Swatinem/rust-cache@v2

      - name: Format check
        run: cargo fmt --all --check

      - name: Clippy
        run: cargo clippy --all-targets -- -D warnings

      - name: Tests
        run: cargo nextest run

      - name: Doc tests
        run: cargo test --doc

      - name: Security audit
        run: cargo audit
```

### Key details

- **`dtolnay/rust-toolchain`** pins the exact Rust version, so the build is reproducible.
- **`Swatinem/rust-cache`** caches the `target/` directory and Cargo registry across runs, cutting build times by 50-80%.
- **`cargo nextest`** runs tests in parallel with better output than `cargo test`. Install it in CI with `cargo binstall cargo-nextest`.

## Continuous Delivery vs Continuous Deployment

These terms are frequently confused.

**Continuous Delivery:** Every change that passes CI is *ready* to deploy, but a human decides when to push the button.

**Continuous Deployment:** Every change that passes CI is *automatically* deployed to production. No human gate.

```
Continuous Delivery:
  Code → CI → [Staging] → Manual approval → [Production]

Continuous Deployment:
  Code → CI → [Staging] → Auto-deploy → [Production]
```

Most teams practice continuous delivery. Continuous deployment requires very high confidence in your test suite, monitoring, and rollback mechanisms.

## Fast CI Strategies

Slow CI kills developer productivity. A 45-minute pipeline means context-switching, stacking PRs, and frustration. Target under 10 minutes for the critical path.

### Parallelization

Split independent checks into separate jobs that run concurrently:

```yaml
jobs:
  fmt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with: { components: rustfmt }
      - run: cargo fmt --all --check

  clippy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with: { components: clippy }
      - uses: Swatinem/rust-cache@v2
      - run: cargo clippy --all-targets -- -D warnings

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo nextest run
```

Format checking finishes in seconds. No reason to block tests on it.

### Dependency caching

The `Swatinem/rust-cache` action caches compiled dependencies. A fresh Rust build may take 8 minutes; with a warm cache it drops to 2 minutes.

### Affected-only testing

In a workspace with many crates, use `cargo nextest run -p <changed-crate>` to only test what changed. Tools like `cargo-workspaces` or CI scripts that compare `git diff` can automate this.

### Larger runners

GitHub Actions offers larger runners (4-core, 8-core, 16-core). A 16-core runner compiling Rust is significantly faster than the default 2-core. The cost increase is often worth the time saved.

## Real-World Examples

### Spotify: 15,000 builds per day

Spotify's CI/CD pipeline processes roughly 15,000 builds per day across hundreds of services. Their approach:

- **Progressive delivery model:** code goes through CI, deploys to a canary environment, gets validated by automated checks and synthetic tests, then gradually rolls out.
- **Golden rule:** every deploy must be independently rollbackable.
- **Squad autonomy:** each team owns its pipeline configuration, but platform teams provide shared CI components for consistency.

### Etsy: Continuous Deployment Pioneer

Etsy adopted continuous deployment in 2010, deploying to production 50+ times per day with ~200 engineers. Key practices:

- Feature flags for incomplete work (code ships to production but is not yet active).
- Comprehensive monitoring dashboards visible to everyone.
- A culture where deploying was normal, not a special event.

They demonstrated that frequent, small deployments are safer than infrequent, large ones -- a principle now backed by DORA research showing elite teams deploy 200x more frequently.

## Common Mistakes

- **No caching.** Every CI run downloads and compiles all dependencies from scratch. Fix: use `rust-cache` or equivalent.
- **Serial pipeline steps.** Format, lint, test, and audit run one after another when they could run in parallel. Fix: split into separate jobs.
- **Flaky tests in CI.** Intermittent failures erode trust in the pipeline. Engineers start ignoring red builds. Fix: quarantine flaky tests, fix them, and track flakiness rate.
- **Skipping CI for "small changes."** The changes that break production are always the ones you were sure were safe. Fix: enforce CI for all PRs with branch protection rules.
- **No security scanning.** `cargo audit` takes seconds and catches known vulnerabilities. There is no reason to skip it.

## Key Takeaways

1. CI is non-negotiable. Every merge to main should be automatically built, linted, tested, and security-scanned.
2. Fast CI matters. Target under 10 minutes. Use caching, parallelization, and larger runners.
3. Pin your toolchain and dependencies for reproducible builds.
4. The Rust compiler is your most powerful CI tool -- if it compiles, entire classes of bugs are already eliminated.
5. Continuous delivery (human gate) is the pragmatic default. Continuous deployment (auto-deploy) requires mature monitoring and rollback.
