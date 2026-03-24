# End-to-End & Performance Testing

## Sub-documents

1. [E2E Testing Strategies](01-e2e-testing-strategies.md) — Critical path testing, the 80/20 rule, test pyramid vs. ice cream cone, environment management, handling flaky tests.
2. [Contract Testing](02-contract-testing.md) — Consumer-driven contracts, Pact workflow, broker architecture, preventing integration failures across microservices.
3. [Mutation Testing](03-mutation-testing.md) — Measuring test quality, mutation operators, mutation score, cargo-mutants, when mutation testing pays off.
4. [Fuzz Testing](04-fuzz-testing.md) — Coverage-guided fuzzing, cargo-fuzz, structured inputs with Arbitrary, corpus management, security bugs found by fuzzing.
5. [Load & Stress Testing](05-load-and-stress-testing.md) — Load testing, stress testing, soak testing, spike testing. Percentiles over averages, k6 examples, Amazon Prime Day prep, Twitter's Fail Whale, performance testing in CI.

## Diagrams

![Contract Testing](diagrams/contract-testing.png)

![Load Test Stages](diagrams/load-test-stages.png)
