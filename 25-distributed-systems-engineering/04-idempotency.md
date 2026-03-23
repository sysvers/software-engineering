# Idempotency

## Overview

In distributed systems, network failures and retries mean that a request may be delivered more than once. An operation is idempotent if performing it multiple times produces the same result as performing it once. Without idempotency, retries cause double charges, duplicate records, and corrupted state. Idempotency is not an optimization -- it is a correctness requirement for any distributed system that mutates state.

---

## Why It Matters

Consider a payment API. A client sends a charge request, the server processes it, but the response is lost due to a network timeout. The client retries. Without idempotency, the customer is charged twice.

This is not an edge case. In production distributed systems:
- Network timeouts are routine
- Load balancers retry failed requests
- Message queues deliver messages at-least-once by default
- Client-side retry logic fires on ambiguous failures

The only safe assumption is that every mutating operation will be attempted more than once.

---

## Naturally Idempotent Operations

Some operations are inherently idempotent without any extra mechanism:

- **SET operations:** `SET balance = 100` produces the same result regardless of how many times it runs. Contrast with `INCREMENT balance BY 10`, which is not idempotent.
- **PUT (upsert):** Creating or replacing a resource at a specific key. `PUT /users/123 { name: "Alice" }` is idempotent.
- **DELETE:** Deleting a resource by ID. The first call deletes it; subsequent calls find nothing to delete (the result is the same: the resource does not exist).

**Non-idempotent operations** require explicit mechanisms:
- `POST /charges` (creates a new charge each time)
- `INCREMENT`, `APPEND`, `ENQUEUE`
- Any operation that allocates a new resource with a server-generated ID

---

## Idempotency Keys

The standard approach for making non-idempotent operations safe is the idempotency key: a client-generated unique identifier (typically a UUID) sent with each request.

**How it works:**
1. Client generates a UUID and includes it as a header (e.g., `Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000`)
2. Server checks if this key has been seen before
3. If yes: return the stored response without re-executing the operation
4. If no: execute the operation, store the result keyed by the idempotency key, return the response

**Design considerations:**
- **Storage:** Idempotency records must be stored durably (database, not just in-memory cache). If the server crashes after processing but before the client receives the response, the retry must still find the stored result.
- **Retention:** Records cannot be kept forever. A retention window (e.g., 24-72 hours) limits storage growth. After expiration, a retry with the same key is treated as a new request.
- **Scope:** The key should be scoped to an operation type and identity. The same UUID used for two different API endpoints should not collide.
- **Atomicity:** Checking for and inserting the idempotency key must be atomic with executing the operation. Otherwise, two concurrent requests with the same key could both pass the check and execute.

---

## Designing Idempotent APIs

### HTTP Method Semantics

Per HTTP specification:
- **GET, HEAD, OPTIONS, DELETE** -- should be idempotent by design
- **PUT** -- should be idempotent (replace the full resource)
- **POST** -- not idempotent by default; requires idempotency keys for safety
- **PATCH** -- depends on the operation; absolute patches (`set field = value`) are idempotent, relative patches (`increment field by 1`) are not

### Request Fingerprinting

Beyond just the idempotency key, some systems also fingerprint the request body. If a client reuses an idempotency key but with a different payload, the server should reject the request (return 422 Unprocessable Entity) rather than silently returning the stored result for a different operation.

### Conditional Writes

An alternative to idempotency keys for database operations:

```sql
-- Idempotent: uses IF NOT EXISTS (Cassandra) or ON CONFLICT (PostgreSQL)
INSERT INTO charges (id, amount, customer_id)
VALUES ('charge_abc', 5000, 'cust_123')
ON CONFLICT (id) DO NOTHING;
```

This is naturally idempotent because the operation succeeds exactly once; retries are no-ops.

---

## Rust Implementation

```rust
use std::collections::HashMap;
use std::time::{Duration, Instant};

/// Stores results of previously processed requests for deduplication.
struct IdempotencyStore<R: Clone> {
    entries: HashMap<String, (R, Instant)>,
    retention: Duration,
}

impl<R: Clone> IdempotencyStore<R> {
    fn new(retention: Duration) -> Self {
        Self {
            entries: HashMap::new(),
            retention,
        }
    }

    /// Check whether a request with this idempotency key was already processed.
    fn get_existing_result(&self, key: &str) -> Option<R> {
        self.entries.get(key).and_then(|(result, created_at)| {
            if created_at.elapsed() < self.retention {
                Some(result.clone())
            } else {
                None
            }
        })
    }

    /// Record the result of a processed request.
    fn record(&mut self, key: String, result: R) {
        self.entries.insert(key, (result, Instant::now()));
    }

    /// Purge entries older than the retention period.
    fn purge_expired(&mut self) {
        self.entries
            .retain(|_, (_, created_at)| created_at.elapsed() < self.retention);
    }
}

/// Middleware-style handler demonstrating idempotent request processing.
struct IdempotentHandler<R: Clone> {
    store: IdempotencyStore<R>,
}

impl<R: Clone> IdempotentHandler<R> {
    fn new(retention: Duration) -> Self {
        Self {
            store: IdempotencyStore::new(retention),
        }
    }

    /// Process a request idempotently. If the key was seen before,
    /// return the stored result. Otherwise, execute the operation and store it.
    fn handle<F>(&mut self, idempotency_key: &str, operation: F) -> R
    where
        F: FnOnce() -> R,
    {
        // Check for existing result first.
        if let Some(existing) = self.store.get_existing_result(idempotency_key) {
            println!("Returning cached result for key: {}", idempotency_key);
            return existing;
        }

        // Execute the operation and store the result.
        let result = operation();
        self.store.record(idempotency_key.to_string(), result.clone());
        result
    }
}
```

---

## Stripe's Approach

Stripe's implementation of idempotency is the industry reference. Key details from their engineering blog and API documentation:

### API Design

- Every mutating API endpoint accepts an `Idempotency-Key` header
- Keys are scoped to a Stripe account (two different accounts can use the same key without collision)
- If a request with the same key and same parameters is retried, Stripe returns the original response
- If a request with the same key but different parameters is sent, Stripe returns a 400 error

### Implementation Details

- **State machine:** Each idempotent request goes through a state machine: `started -> processing -> completed` (or `errored`). The key is inserted atomically at the `started` state.
- **Atomic phases:** The processing of a request is broken into "atomic phases" -- each phase is a database transaction. If the server crashes between phases, it can resume from the last completed phase rather than re-executing from the beginning.
- **Recovery:** On retry, Stripe checks the state of the idempotency record. If `completed`, it returns the stored response. If `started` or `processing`, it resumes from where it left off (or waits for the in-flight request to finish).
- **Retention:** Idempotency keys are retained for 24 hours. After that, the same key is treated as a new request.

### Why This Matters for Payments

Payment processing has an asymmetric failure cost. Charging a customer twice is far worse than failing to charge at all (a failed charge can be retried; a double charge requires a refund and erodes trust). Stripe's idempotency design ensures that the worst case for a retry is returning the original result, never re-executing the charge.

### Lessons for Your Own APIs

1. **Make idempotency keys mandatory for payment and financial APIs**, not optional.
2. **Use database transactions** to atomically check-and-insert the idempotency key. Race conditions between two concurrent retries must be resolved by the database, not application logic.
3. **Store the full response**, not just a success/failure flag. The client expects the exact same response on a retry.
4. **Distinguish between retryable and non-retryable failures.** A 500 error means "try again"; a 400 error means "do not retry with the same parameters."

---

## Common Pitfalls

1. **Storing idempotency keys only in memory:** If the server restarts, all keys are lost and retries create duplicates. Use durable storage.

2. **Non-atomic check-and-execute:** If two requests with the same key arrive simultaneously and both pass the "key not found" check before either inserts, both execute. Use database-level uniqueness constraints or distributed locks.

3. **Forgetting to fingerprint the request body:** An idempotency key reused with different parameters should fail loudly, not silently return a stale response for a different operation.

4. **Treating idempotency as optional:** If an API is not idempotent, every caller must implement exactly-once delivery logic. This is nearly impossible in a distributed system. Make the server idempotent so callers can safely retry.

5. **Not purging expired keys:** Idempotency stores grow without bound if expired entries are not cleaned up. Run periodic purges or use database TTL features (e.g., PostgreSQL `pg_cron`, Redis `EXPIRE`).

---

## Key Takeaways

- Every mutating API in a distributed system must be idempotent. This is a correctness requirement, not a nice-to-have.
- Idempotency keys (client-generated UUIDs) are the standard mechanism for non-naturally-idempotent operations.
- The check-and-insert of the idempotency key must be atomic with the operation execution to prevent race conditions.
- Store the full response alongside the idempotency key so retries return identical results.
- Stripe's approach (state machine, atomic phases, 24-hour retention) is the gold standard for payment systems and a solid model for any critical API.
