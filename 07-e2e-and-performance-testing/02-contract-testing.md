# Contract Testing

## The Problem Contract Testing Solves

In a microservice architecture, services communicate over the network. Each service has its own team, its own repository, and its own deployment pipeline. This independence is the point of microservices — but it creates a dangerous gap.

### The Classic Integration Failure

```
Service A (OrderService) calls Service B (UserService) API:
  GET /users/123 → expects { id, email, name }

Service B's team renames "name" to "full_name" in a refactor.

Service B's unit tests: PASS (code is internally correct)
Service A's unit tests: PASS (it's mocking Service B's response)
Integration in production: BROKEN
```

Both services pass their own tests. The contract between them was violated, and nobody noticed until production broke.

This is the fundamental problem: **mocks drift from reality**. The mock in Service A assumes a response shape that Service B no longer provides.

---

## What Is Contract Testing?

Contract testing verifies that two services (a **consumer** and a **provider**) agree on the shape of their communication — without testing the full E2E flow.

It is faster than E2E testing, more reliable than mocks, and catches exactly the class of bugs described above.

### Consumer-Driven Contracts

The dominant approach is **consumer-driven contracts (CDC)**. The consumer defines what it needs from the provider, and the provider verifies it can fulfill those needs.

Why consumer-driven? Because the consumer knows what it actually uses. A provider API might return 30 fields, but a given consumer may only use 3. The contract captures only what matters to the consumer.

---

## How Pact Works

[Pact](https://pact.io/) is the most widely used contract testing framework. It supports many languages (JavaScript, Java, Python, Ruby, Go, Rust, .NET).

### The Workflow

```
┌──────────┐    generates    ┌──────────┐    verifies    ┌──────────┐
│ Consumer │ ──────────────→ │  Pact    │ ←──────────── │ Provider │
│  Tests   │    contract     │  Broker  │    against     │  Tests   │
└──────────┘                 └──────────┘    contract     └──────────┘
```

**Step 1: Consumer writes a test defining its expectations**

```javascript
// OrderService (consumer) test
const { Pact } = require('@pact-foundation/pact');

const provider = new Pact({
  consumer: 'OrderService',
  provider: 'UserService',
});

describe('UserService API', () => {
  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());

  it('returns user details', async () => {
    // Define the expected interaction
    await provider.addInteraction({
      state: 'user 123 exists',
      uponReceiving: 'a request for user 123',
      withRequest: {
        method: 'GET',
        path: '/users/123',
        headers: { Accept: 'application/json' },
      },
      willRespondWith: {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: {
          id: 123,
          email: 'alice@example.com',
          name: 'Alice',   // Consumer depends on this field
        },
      },
    });

    // Run the consumer code against the mock
    const user = await userClient.getUser(123);
    expect(user.name).toBe('Alice');
  });
});
```

This test generates a **contract file** (a Pact JSON file):

```json
{
  "consumer": { "name": "OrderService" },
  "provider": { "name": "UserService" },
  "interactions": [
    {
      "description": "a request for user 123",
      "providerState": "user 123 exists",
      "request": {
        "method": "GET",
        "path": "/users/123",
        "headers": { "Accept": "application/json" }
      },
      "response": {
        "status": 200,
        "headers": { "Content-Type": "application/json" },
        "body": {
          "id": 123,
          "email": "alice@example.com",
          "name": "Alice"
        }
      }
    }
  ]
}
```

**Step 2: Contract is published to a Pact Broker**

```bash
pact-broker publish ./pacts \
  --consumer-app-version=$(git rev-parse HEAD) \
  --broker-base-url=https://your-broker.example.com
```

**Step 3: Provider verifies the contract**

```javascript
// UserService (provider) verification test
const { Verifier } = require('@pact-foundation/pact');

describe('Pact verification', () => {
  it('fulfills OrderService contract', async () => {
    const verifier = new Verifier({
      providerBaseUrl: 'http://localhost:3000',
      pactBrokerUrl: 'https://your-broker.example.com',
      provider: 'UserService',
      providerVersion: process.env.GIT_SHA,
      stateHandlers: {
        'user 123 exists': async () => {
          // Set up the required state in the provider
          await db.users.create({ id: 123, email: 'alice@example.com', name: 'Alice' });
        },
      },
    });

    await verifier.verifyProvider();
  });
});
```

If UserService removes the `name` field, the verification **fails in CI** — before deployment. The breaking change is caught before it reaches production.

---

## The Pact Broker and "Can I Deploy?"

The Pact Broker is a central service that:

- Stores all contracts between consumers and providers
- Tracks which versions are compatible
- Provides a **"Can I Deploy?"** check for CI/CD pipelines

```bash
# Before deploying UserService, check if it's compatible
# with all its consumers
pact-broker can-i-deploy \
  --pacticipant=UserService \
  --version=$(git rev-parse HEAD) \
  --to-environment=production
```

This returns a pass/fail based on whether all consumer contracts are satisfied. It becomes a **deployment gate** — services cannot deploy if they break a consumer's contract.

---

## Real-World Example: Atlassian

Atlassian adopted Pact contract testing across their microservice ecosystem. Before contract testing:

- A breaking API change in one service could cascade across dozens of dependent services
- Production incidents from integration failures took **days to diagnose** because the root cause was a subtle field change weeks earlier
- Teams were afraid to change APIs, slowing development

After adopting Pact:

- **Integration failures dropped by 80%** in the first quarter
- Breaking changes are caught in CI within minutes, not discovered in production days later
- Teams deploy independently with confidence — the broker tells them if their changes are safe
- The investment paid for itself within the first quarter through reduced incident costs and engineering time

### What Made It Work at Atlassian

1. **Mandatory broker checks** — no service could deploy without passing `can-i-deploy`
2. **Provider state management** — providers maintained proper state setup for contract tests
3. **Versioned contracts** — contracts were tied to Git SHAs, enabling precise compatibility tracking
4. **Cultural adoption** — teams were trained on writing good contracts, not just given the tool

---

## Contract Testing vs. E2E Testing

| Dimension | Contract Testing | E2E Testing |
|-----------|-----------------|-------------|
| **Speed** | Seconds (runs against mocks/stubs) | Minutes to hours |
| **What it tests** | Shape of communication between services | Full user-visible behavior |
| **Flakiness** | Very low (no network, no UI) | High (timing, state, network) |
| **Cost** | Low (lightweight, runs in CI) | High (needs full environment) |
| **What it misses** | Business logic, full behavior | Nothing in scope, but expensive |

Contract testing is **not a replacement** for E2E testing. It catches a specific, common class of bugs (interface mismatches) very efficiently. E2E tests catch a broader class of bugs (full behavior) but at much higher cost.

Use both. Use contract tests for all service-to-service boundaries. Use E2E tests only for critical business flows.

---

## Common Pitfalls

### Over-specifying Contracts

```json
// Bad: specifying exact values unnecessarily
"body": { "id": 123, "email": "alice@example.com", "created_at": "2024-01-15T10:30:00Z" }

// Good: using matchers for dynamic fields
"body": {
  "id": { "pact:matcher:type": "integer", "value": 123 },
  "email": { "pact:matcher:type": "regex", "regex": ".+@.+", "value": "alice@example.com" },
  "created_at": { "pact:matcher:type": "datetime", "format": "yyyy-MM-dd'T'HH:mm:ss'Z'" }
}
```

Over-specified contracts break on irrelevant changes (a different example email) and miss real issues (email field changed to an integer).

### Not Managing Provider States

Contract tests require the provider to be in a specific state (e.g., "user 123 exists"). If provider state setup is neglected, verification tests become unreliable.

### Treating Contracts as Integration Tests

Contracts should test the **shape** of communication (fields, types, status codes), not business logic. Don't try to verify that a discount calculation is correct through a contract — that's a unit test.

### Forgetting to Update Contracts

When a consumer starts using a new field, the contract must be updated. Stale contracts give false confidence.

---

## When to Use Contract Testing

**Use for:**
- Microservice architectures with many service-to-service calls
- Teams that deploy independently on different schedules
- APIs consumed by external partners or third parties
- Any boundary where two teams own different sides

**Avoid for:**
- Monoliths (no service boundaries to test)
- Simple applications with one or two services (E2E tests are sufficient)
- Testing business logic (use unit tests)

---

## Key Takeaways

1. Contract testing catches the #1 cause of microservice integration failures: interface mismatches.
2. Consumer-driven contracts ensure providers don't break what consumers actually use.
3. The Pact Broker's "Can I Deploy?" check turns contract testing into an automated deployment gate.
4. Contracts test shape, not behavior. Use matchers for dynamic fields, not exact values.
5. Atlassian's experience shows contract testing pays for itself quickly in reduced incidents and debugging time.
