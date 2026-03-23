# Load & Stress Testing

## Why Performance Testing Matters

Your application works correctly. Every unit test passes, every integration test is green, every E2E flow succeeds. Then you launch, 10,000 users hit the system simultaneously, and it falls over.

Functional correctness and performance under load are entirely different properties. A function that returns the right answer in 2ms for one user might take 30 seconds when 1,000 users call it concurrently because it's holding a database lock, exhausting a connection pool, or triggering garbage collection storms.

Performance testing verifies that your system meets its performance requirements **under realistic conditions** — before users discover that it doesn't.

---

## Types of Performance Tests

| Type | Goal | Load Pattern | Duration |
|------|------|--------------|----------|
| **Load test** | Verify system handles expected traffic | Normal expected traffic | 10-60 minutes |
| **Stress test** | Find the breaking point | Gradually increasing beyond capacity | 30-60 minutes |
| **Soak test** | Find leaks and degradation over time | Sustained normal load | 4-24 hours |
| **Spike test** | Verify handling of sudden surges | Sudden burst, then back to normal | 10-30 minutes |
| **Capacity test** | Determine maximum throughput | Increase until SLAs are violated | Variable |

Each type answers a different question. You need all of them at different stages of your system's lifecycle.

---

## What to Measure: Percentiles, Not Averages

The single most important rule of performance measurement: **never use averages as your primary metric**. Averages hide the experience of your worst-off users.

### Why Averages Lie

```
Average response time: 200ms   <-- Looks great!
p50 (median):         150ms   <-- Half of users see this
p95:                  800ms   <-- 5% of users wait this long
p99:                  3,200ms <-- 1% of users wait 3+ seconds
```

If you have 1 million daily users, a p99 of 3.2 seconds means **10,000 users per day** have a terrible experience. The average hides this completely because the fast majority drowns out the slow minority.

### The Percentiles That Matter

- **p50 (median)** — the typical experience. Useful for general health monitoring.
- **p95** — the experience of your unlucky-but-not-rare users. This is the primary metric most teams should set SLAs against.
- **p99** — the tail. If this is bad, a meaningful number of users are suffering. At scale, 1% is a lot of people.
- **p99.9** — relevant for high-scale systems. At Amazon's scale, p99.9 affects millions of requests per day.

### Key Metrics to Track

| Metric | What it tells you |
|--------|-------------------|
| **Response time (p50/p95/p99)** | How fast the system responds under load |
| **Throughput (req/s)** | How many requests the system can handle |
| **Error rate** | What percentage of requests fail under load |
| **CPU utilization** | Whether compute is the bottleneck |
| **Memory usage** | Whether memory is leaking or growing |
| **Connection pool usage** | Whether database or HTTP connections are saturated |
| **Queue depth** | Whether async work is backing up |

---

## Load Testing

Load testing simulates real-world traffic at expected volumes. The goal is to verify that your system meets its performance requirements (SLAs) under normal conditions.

### Load Test with k6

[k6](https://k6.io/) is a modern load testing tool where you write test scripts in JavaScript. It's designed for developer workflows and CI integration.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 virtual users
    { duration: '5m', target: 100 },  // Hold at 100 users for 5 minutes
    { duration: '2m', target: 0 },    // Ramp down to 0
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests must complete in < 500ms
    http_req_failed: ['rate<0.01'],    // Less than 1% failure rate
  },
};

export default function () {
  const res = http.get('https://api.example.com/products');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time OK': (r) => r.timings.duration < 500,
  });
  sleep(1); // Simulate user think time
}
```

The `thresholds` block is critical: k6 exits with a non-zero code if thresholds are violated, which means **CI fails on performance regressions**.

### Realistic Load Profiles

Real traffic doesn't follow uniform patterns. A good load test models actual user behavior — browsing (100% of users), adding to cart (30%), checking out (10%). Use k6's `group()` to label each stage of the funnel and `Math.random()` to branch users into different paths with realistic probabilities. This prevents the common mistake of load testing only a single endpoint uniformly.

---

## Stress Testing

Stress testing pushes the system **beyond its limits** to answer two questions: (1) at what point does the system break, and (2) how does it break?

### What You Want to Learn

- **Degradation point** — at what load does p95 latency exceed acceptable thresholds?
- **Breaking point** — at what load do errors start occurring?
- **Failure mode** — does the system degrade gracefully or collapse catastrophically?
- **Recovery** — when load drops, does the system recover on its own?
- **Bottleneck** — which component fails first (database, API server, cache, network)?

### Graceful Degradation vs. Cascading Failure

```
Graceful:  Load increases -> Responses slow -> Some requests shed -> System stays up
Cascade:   Load increases -> Service A slows -> Service B retries -> Service A overwhelmed -> Both crash
```

Cascading failures are the most dangerous outcome of insufficient stress testing. They are caused by retry storms, missing circuit breakers, and shared resource pools. Stress tests reveal them before production does.

### Stress Test with k6

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Normal load
    { duration: '5m', target: 500 },   // 5x normal
    { duration: '5m', target: 1000 },  // 10x normal
    { duration: '5m', target: 2000 },  // 20x normal -- where does it break?
    { duration: '5m', target: 0 },     // Ramp down -- does it recover?
  ],
};
```

The ramp-down phase is as important as the ramp-up. A system that doesn't recover after load decreases has a serious problem (leaked connections, stuck threads, exhausted resources that aren't released).

---

## Soak Testing

Soak tests (endurance tests) run at normal load for an extended period — hours or days — to find problems that only manifest over time.

### What Soak Tests Find

- **Memory leaks** — memory usage climbs slowly until the process is killed by the OOM killer
- **Connection pool exhaustion** — connections acquired but never returned
- **File descriptor leaks** — handles accumulate until the OS limit is hit
- **Database connection drift** — connections go stale, error rate creeps up
- **Log file growth** — logs fill the disk, the process crashes on write
- **GC pressure** — garbage collection pauses grow longer as heap fragments

### A Soak Test Looks Unremarkable Until It Doesn't

At the 1-hour mark, everything looks fine. At the 4-hour mark, memory is up 15%. At the 8-hour mark, p99 latency has doubled. At the 12-hour mark, the process is consuming 90% of available memory and GC pauses are causing request timeouts.

This is why you monitor trends over the full duration, not just the final pass/fail. Export metrics to a dashboard (Grafana, Datadog) and look for upward slopes in resource consumption.

---

## Spike Testing

Spike tests simulate sudden, dramatic traffic surges — the kind that happen during product launches, marketing campaigns going viral, breaking news events, or (in the case of e-commerce) flash sales.

### Spike Test with k6

```javascript
export const options = {
  stages: [
    { duration: '1m', target: 50 },    // Normal traffic
    { duration: '10s', target: 1000 },  // Sudden spike to 1000 users
    { duration: '3m', target: 1000 },   // Hold the spike
    { duration: '10s', target: 50 },    // Spike ends abruptly
    { duration: '3m', target: 50 },     // Recovery period
  ],
};
```

The key questions: How long does the system take to absorb the spike? Does auto-scaling kick in fast enough? Do connection pools saturate during the ramp? Does the system settle back to normal performance after the spike ends?

---

## Real-World Lessons

### Amazon's Prime Day Preparation

Amazon runs "GameDay" exercises — simulated Prime Day-level traffic against their production systems weeks before the actual event. Key practices:

- **Test at 2-3x expected peak** — this provides headroom for traffic that exceeds projections
- **Failure injection during load** — they kill services, overwhelm databases, and simulate network partitions while under load, combining stress testing with chaos engineering
- **Progressive rollout** — they test individual services under load first, then full system integration
- **p99.9 thresholds** — at Amazon's scale, even p99.9 affects millions of requests, so they optimize tail latency aggressively

This preparation is why Prime Day handles hundreds of millions of transactions without major outages. The investment in testing infrastructure is enormous, but the cost of a 1-hour Prime Day outage dwarfs it.

### Twitter's Fail Whale Era

In its early years (2007-2012), Twitter experienced frequent outages under traffic spikes — the infamous "Fail Whale" error page. Root causes:

- **Insufficient load testing** — they hadn't stress tested at the scale of real-world events (Super Bowl tweets, celebrity death announcements, election nights)
- **Monolithic architecture** — a single Ruby on Rails application couldn't scale horizontally
- **No graceful degradation** — when load exceeded capacity, the entire system went down rather than shedding non-critical work

Twitter's response was a multi-year investment: rewriting core services in Scala/Java, adopting a microservice architecture, and building comprehensive load testing infrastructure. They eventually evolved to handle 150,000+ tweets per second during peak events. The Fail Whale era illustrates what happens when you discover your performance limits in production instead of in testing.

---

## Performance Testing in CI

Performance testing is not a one-time event before launch. Performance regressions are introduced constantly — a new ORM query that does N+1 selects, an added middleware that serializes a large object, an updated dependency that's slower. Running performance tests in CI catches these regressions before they reach production.

### Integrating k6 with CI

```yaml
# GitHub Actions example
name: Performance Tests
on: [push, pull_request]

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Start application
        run: docker compose up -d --wait
      - name: Run load test
        run: k6 run tests/load-test.js
      - name: Tear down
        if: always()
        run: docker compose down -v
```

Because k6 thresholds cause a non-zero exit code on failure, the CI step fails automatically when performance degrades beyond defined limits.

### What to Test in CI vs. Dedicated Runs

| In CI (every PR) | Dedicated runs (scheduled) |
|-------------------|---------------------------|
| Smoke load test (low user count, short duration) | Full load test at production-scale traffic |
| Key endpoint latency thresholds | Stress test to find breaking point |
| Regression detection (compare to baseline) | Soak test for memory leaks |
| < 5 minutes total | Spike test for surge resilience |

CI tests should be fast (under 5 minutes) and focused on detecting regressions. Full-scale load, stress, soak, and spike tests run on a schedule (nightly or weekly) against a staging environment that mirrors production.

---

## Common Pitfalls

### Testing Against Unrealistic Environments

Running a load test against a single-node staging server and extrapolating to a 10-node production cluster is meaningless. Your bottlenecks will be completely different — staging might be CPU-bound on a single core while production might be database-connection-bound across ten nodes. Test against an environment that mirrors production topology.

### Testing Only the Happy Path

What happens when the database is slow? When a downstream service returns 500s? When the cache is cold? Load testing only the sunny-day scenario misses the most dangerous failure modes. Combine load testing with fault injection for realistic results.

### No Baseline, No Comparison

A p95 of 450ms means nothing without context. Is that better or worse than last week? Set baselines and track trends. Many teams store performance results in a time-series database and alert on degradation.

---

## Key Takeaways

1. Always measure p95 and p99 latency, never just averages. Averages hide the experience of your most frustrated users.
2. Load tests verify expected traffic. Stress tests find the breaking point. Soak tests find leaks. Spike tests verify surge handling. You need all four.
3. The ramp-down phase of a stress test matters as much as the ramp-up. A system that doesn't recover has a serious problem.
4. Performance testing in CI catches regressions before production. Use lightweight smoke tests on every PR and full-scale tests on a schedule.
5. Test against environments that mirror production topology. Single-node staging results don't predict multi-node production behavior.
6. Combine load testing with failure injection. Real production traffic doesn't arrive in a vacuum — it arrives while things are going wrong.
7. Soak tests are boring by design and invaluable in practice. The memory leak that takes 8 hours to manifest will crash your service at 3 AM on a Saturday.
