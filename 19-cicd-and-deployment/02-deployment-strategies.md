# Deployment Strategies

## Why Deployment Strategy Matters

Deployment strategy determines how much risk you take each time you ship code to production. The wrong strategy turns every deploy into a potential outage. The right one makes deployments boring -- and boring is exactly what you want.

## Blue-Green Deployment

Maintain two identical production environments. Deploy to the inactive one, verify it works, then switch traffic.

```
Before:  Traffic → [Blue (v1)] ← active     [Green (idle)]
Deploy:  Traffic → [Blue (v1)] ← active     [Green (v2)] ← deploying
Switch:  Traffic → [Blue (idle)]             [Green (v2)] ← active
```

### How it works

1. **Blue** is running v1, serving all production traffic.
2. Deploy v2 to **Green** (which receives no traffic).
3. Run smoke tests against Green.
4. Switch the load balancer / DNS to point at Green.
5. Green is now live. Blue becomes the idle standby.

### Rollback

Instant -- switch traffic back to Blue. The old version is still running, untouched.

### When to use

- Services where instant rollback is critical (payment processing, authentication).
- Stateless services. Stateful services are harder because database schema changes may not be backward-compatible.
- When you can afford double infrastructure cost (or use cloud auto-scaling to minimize it).

### Trade-offs

| Pros | Cons |
|------|------|
| Zero downtime | Double infrastructure cost during deployment |
| Instant rollback (seconds) | Database migrations need careful handling |
| Full environment validation before switch | DNS propagation can delay the switch (if DNS-based) |

## Canary Deployment

Route a small percentage of traffic to the new version. Monitor for errors. Gradually increase if healthy. Roll back automatically if not.

```
Step 1:   5% traffic → [v2]    95% traffic → [v1]
Step 2:  25% traffic → [v2]    75% traffic → [v1]
Step 3:  50% traffic → [v2]    50% traffic → [v1]
Step 4: 100% traffic → [v2]     0% traffic → [v1]
```

### How it works

1. Deploy v2 alongside v1.
2. Route 5% of traffic to v2.
3. Monitor error rates, latency, and business metrics for a defined period (e.g., 10 minutes).
4. If metrics are healthy, increase to 25%, then 50%, then 100%.
5. If any step shows degradation, automatically route 100% back to v1.

### Automated canary analysis

The key to effective canary deployments is automated metric comparison:

```
Canary metrics during window:
  - Error rate:  canary 0.12%  vs  baseline 0.10%  → PASS (within threshold)
  - p99 latency: canary 180ms  vs  baseline 165ms  → PASS (within threshold)
  - CPU usage:   canary 45%    vs  baseline 40%    → PASS (within threshold)

Verdict: PROMOTE to next stage
```

Without automated analysis, a canary deployment is just a deployment with extra steps. You need metrics and automated comparison to detect regressions.

### When to use

- User-facing services where regressions directly impact revenue.
- Services with good observability (metrics, tracing, alerting).
- When you want real production traffic validation, not just staging tests.

### Trade-offs

| Pros | Cons |
|------|------|
| Limited blast radius (only 5% of users affected initially) | More complex routing infrastructure |
| Real production validation | Requires good metrics and automated analysis |
| Gradual confidence building | Slower rollout than blue-green |

## Rolling Deployment

Update instances one at a time. Each instance is taken out of the load balancer, updated, health-checked, and returned to service.

```
[v1] [v1] [v1] [v1]    ← Start
[v2] [v1] [v1] [v1]    ← First instance updated
[v2] [v2] [v1] [v1]    ← Second instance updated
[v2] [v2] [v2] [v1]    ← Third instance updated
[v2] [v2] [v2] [v2]    ← Complete
```

### How it works

1. Remove one instance from the load balancer.
2. Deploy v2 to that instance.
3. Run health checks against it.
4. If healthy, add it back to the load balancer and proceed to the next instance.
5. If unhealthy, stop the rollout and roll back the updated instances.

### When to use

- Internal services where brief version mixing is acceptable.
- When you cannot afford double infrastructure (blue-green cost).
- Kubernetes uses rolling deployments by default (`strategy: RollingUpdate`).

### Trade-offs

| Pros | Cons |
|------|------|
| No extra infrastructure needed | Mixed versions during deployment |
| Gradual rollout | Slower rollback (must re-deploy all instances) |
| Built into Kubernetes natively | API compatibility required between v1 and v2 |

## Automated Rollback

Regardless of strategy, every deployment needs an automated rollback mechanism. Manual rollbacks under pressure at 2 AM are where mistakes compound.

### Rollback triggers

- **Error rate exceeds threshold.** If the 5xx error rate jumps from 0.1% to 2%, roll back.
- **Latency spike.** If p99 latency doubles compared to the pre-deploy baseline, roll back.
- **Health check failure.** If the new version fails readiness probes, Kubernetes rolls back automatically.
- **Business metric anomaly.** If checkout conversion drops 10%, roll back (even if technical metrics look fine).

### Rollback speed by strategy

| Strategy | Rollback speed | Mechanism |
|----------|---------------|-----------|
| Blue-green | Seconds | Switch traffic back to old environment |
| Canary | Seconds | Route 100% back to v1 |
| Rolling | Minutes | Re-deploy v1 across all instances |

## Real-World Examples

### GitHub: ChatOps Deployment

GitHub deploys to production dozens of times per day using a ChatOps system:

1. An engineer types `/deploy` in a chat channel.
2. The system triggers a canary deployment.
3. Automated monitoring watches error rates and latency for 10 minutes.
4. If healthy, it proceeds to full rollout.
5. If not, it automatically rolls back.
6. The entire process takes ~15 minutes from command to full deployment.

The ChatOps approach gives visibility: everyone in the channel can see who is deploying what, and the bot reports status updates in real time.

### Amazon: 11.7-Second Deploys

Amazon deploys code every 11.7 seconds on average (across all services). This velocity is enabled by:

- Fully automated CI/CD pipelines with no manual gates.
- Automated canary analysis comparing metrics against baselines.
- **One-box deployment:** deploy to a single instance first, validate, then proceed.
- Automatic rollback triggered by metric anomalies.

Their deployment philosophy: small changes, deployed frequently, are safer than large changes deployed rarely. Each deployment touches a small amount of code, so when something breaks, the cause is obvious.

### Etsy: Deploying 50+ Times Per Day

Etsy pioneered continuous deployment in 2010 with ~200 engineers deploying 50+ times per day:

- Feature flags decouple deployment from release (code ships but is not active).
- Comprehensive monitoring dashboards visible to the entire company.
- Deploying was normal -- not a special event requiring a change advisory board.

## Choosing a Strategy

| Scenario | Recommended strategy |
|----------|---------------------|
| Payment / auth service, instant rollback critical | Blue-green |
| User-facing API with good observability | Canary |
| Internal service, cost-sensitive | Rolling |
| Database migration (schema change) | Blue-green with backward-compatible migrations |
| First deployment of a new service | Rolling (simplest) |

## Common Mistakes

- **Canary without automated metrics.** If nobody is watching the canary, it is just a deployment with extra steps.
- **No rollback testing.** A rollback plan that has never been tested is not a plan. Test rollbacks regularly.
- **Deploying on Friday.** Unless your monitoring and on-call are excellent, avoid deploying before weekends.
- **Manual deployments.** Any manual step is a step that can be forgotten, done incorrectly, or skipped under pressure. Automate everything.
- **Large, infrequent deploys.** Deploying 10,000 lines of changes is riskier than deploying 100 lines ten times. Small batches reduce blast radius.

## Key Takeaways

1. Deployment strategy should match risk tolerance. Canary for user-facing, rolling for internal, blue-green for critical.
2. Automated rollback is not optional. Define triggers (error rate, latency, health checks) and test them.
3. Small, frequent deployments are safer than large, infrequent ones. This is counter-intuitive but consistently proven by DORA research.
4. Always have a rollback plan. Test it before you need it.
