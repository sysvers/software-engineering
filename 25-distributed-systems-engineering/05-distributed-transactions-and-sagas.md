# Distributed Transactions and Sagas

## Overview

When a business operation spans multiple services or databases, ensuring atomicity (all-or-nothing) becomes a fundamental challenge. Traditional ACID transactions work within a single database, but in a microservice architecture, each service owns its own data store. Distributed transactions and sagas are the two primary approaches to maintaining data consistency across service boundaries.

---

## Two-Phase Commit (2PC)

The classic protocol for distributed atomicity. A coordinator manages the transaction across multiple participants.

### How It Works

**Phase 1 -- Prepare:**
1. Coordinator sends `PREPARE` to all participants
2. Each participant writes the transaction to a durable log (WAL) and locks the affected resources
3. Each participant votes `YES` (ready to commit) or `NO` (cannot commit)

**Phase 2 -- Commit/Abort:**
- If all participants voted `YES`: coordinator sends `COMMIT`; participants finalize and release locks
- If any participant voted `NO`: coordinator sends `ABORT`; participants roll back and release locks

### Why 2PC Is Problematic

- **Blocking protocol:** If the coordinator crashes after sending `PREPARE` but before sending `COMMIT`, participants hold locks indefinitely, waiting for a decision that may never come. This blocks all other transactions touching those rows.
- **Single point of failure:** The coordinator is critical. If it dies at the wrong moment, manual intervention is required to resolve in-doubt transactions.
- **High latency:** Two round-trips plus durable logging on every participant makes 2PC slow, especially across datacenters.
- **Tight coupling:** All participants must be available simultaneously. A single slow or failed participant blocks the entire transaction.

### When 2PC Is Still Used

Despite its flaws, 2PC appears in:
- **Databases with XA transactions** (PostgreSQL, MySQL, Oracle) -- within a single datacenter and with a reliable coordinator
- **Google Spanner** -- uses 2PC within Paxos groups, but Paxos handles coordinator failure, making it a non-blocking variant
- **Message broker integration** -- some systems use 2PC between a database and a message broker for exactly-once publishing

---

## The Saga Pattern

Sagas replace distributed transactions by breaking them into a sequence of local transactions, each with a corresponding compensating action. If any step fails, the saga executes compensations in reverse order to undo the work of previously completed steps.

### Choreography

Each service publishes domain events, and downstream services react. There is no central coordinator.

```
OrderService           PaymentService          InventoryService
    |                       |                        |
    |-- OrderCreated ------>|                        |
    |                       |-- PaymentCharged ----->|
    |                       |                        |-- InventoryReserved
    |                       |                        |
    (if InventoryReserved fails)                     |
    |                       |<-- ReservationFailed --|
    |<-- PaymentRefunded ---|                        |
    |-- OrderCancelled      |                        |
```

**Advantages:**
- Loose coupling between services
- No single point of failure
- Each service only knows about its own events

**Disadvantages:**
- Harder to understand the overall flow -- the process is implicit in the event chain
- Difficult to debug and monitor -- there is no single place that shows the saga's state
- Cyclic dependencies can emerge if event chains become complex

### Orchestration

A central saga orchestrator directs each step and triggers compensations on failure.

```
SagaOrchestrator
    |
    |--> OrderService.createOrder()        --> OK
    |--> PaymentService.charge()           --> OK
    |--> InventoryService.reserve()        --> FAILED
    |
    |--> PaymentService.refund()           (compensate)
    |--> OrderService.cancelOrder()        (compensate)
```

**Advantages:**
- Explicit, readable control flow
- Easy to monitor saga state (the orchestrator tracks it)
- Easier to test -- the orchestrator can be tested independently

**Disadvantages:**
- Orchestrator is a coordination bottleneck
- Risk of the orchestrator becoming a "god service" that knows too much about other services' internals

---

## Rust Implementation

### Saga Orchestrator

```text
/// Represents a single step in a saga with its action and compensation.
STRUCTURE SagaStep
    name : String
    execute : Function → Result        // Returns Ok on success, Error(reason) on failure
    compensate : Function → Result     // Compensating action to undo this step

/// An orchestration-based saga executor.
STRUCTURE SagaOrchestrator
    steps : List<SagaStep>

/// Execute the saga. On failure, compensate all completed steps in reverse.
PROCEDURE SagaOrchestrator.EXECUTE() → Result
    completed ← EMPTY LIST

    FOR i ← 0 TO LENGTH(self.steps) - 1 DO
        step ← self.steps[i]
        PRINT "Executing step: " + step.name
        result ← step.EXECUTE()
        IF result IS OK THEN
            APPEND i TO completed
        ELSE
            reason ← result.ERROR
            PRINT "Step '" + step.name + "' failed: " + reason + ". Initiating compensation."
            FOR EACH idx IN REVERSE(completed) DO
                comp_step ← self.steps[idx]
                PRINT "Compensating step: " + comp_step.name
                comp_result ← comp_step.COMPENSATE()
                IF comp_result IS ERROR THEN
                    PRINT "Compensation for '" + comp_step.name + "' failed: "
                          + comp_result.ERROR + ". Manual intervention required."
                END IF
            END FOR
            RETURN Error("Saga failed at step '" + step.name + "': " + reason)
        END IF
    END FOR

    RETURN Ok
```

### Saga with Persistent State

In production, the orchestrator must persist its state so it can recover after a crash:

```text
/// Tracks the persistent state of a saga execution.
ENUMERATION SagaState
    /// Saga is executing forward steps.
    Running { completed_steps : List<String> }
    /// A step failed; compensations are in progress.
    Compensating {
        failed_step : String,
        compensated_steps : List<String>,
        remaining_compensations : List<String>
    }
    /// All steps completed successfully.
    Completed
    /// All compensations completed (or saga was fully rolled back).
    Aborted { reason : String }
    /// A compensation failed -- manual intervention required.
    CompensationFailed {
        failed_step : String,
        failed_compensation : String,
        error : String
    }

/// A saga execution record that would be persisted to a database.
STRUCTURE SagaExecution
    saga_id : String
    saga_type : String
    state : SagaState
    created_at : Timestamp
    updated_at : Timestamp
```

---

## Compensating Actions

Compensations are not the same as "undo." They are semantic reversals that account for the fact that other things may have happened since the original action.

### Design Principles

1. **Compensations must be idempotent.** The compensation itself might need to be retried if it fails transiently.

2. **Compensations cannot always restore the original state.** If you sent an email notification, you cannot "unsend" it. The compensation might be sending a "correction" email. If you charged a credit card, the compensation is a refund (which is a new transaction, not a rollback).

3. **Compensations should be commutative where possible.** The order of compensations should not matter when steps are independent.

4. **Log everything.** Compensations are the recovery path. If they fail, an operator needs to manually resolve the situation. Detailed logs are essential.

### Compensation Examples

| Forward Action | Compensation |
|---|---|
| Create order | Cancel order (set status to CANCELLED) |
| Charge payment | Issue refund |
| Reserve inventory | Release reservation |
| Send confirmation email | Send cancellation email |
| Allocate resource | Deallocate resource |

---

## Real Failure Scenarios and Solutions

### Scenario 1: Payment Charged but Inventory Unavailable

**Flow:** Order created -> Payment charged -> Inventory check fails (out of stock)

**Without saga:** Customer is charged, but they do not get their item. Manual refund needed.

**With saga (orchestration):**
1. Orchestrator detects inventory failure
2. Orchestrator calls `PaymentService.refund()`
3. Orchestrator calls `OrderService.cancel()`
4. Customer sees "Order could not be completed" and is not charged

**Complication:** What if the refund fails? The orchestrator marks the saga as `CompensationFailed` and alerts operations for manual intervention. This is why compensation failure handling must be designed, not ignored.

### Scenario 2: Network Partition During Saga Execution

**Flow:** Order created -> Payment charged -> Network partition isolates InventoryService

**Problem:** The orchestrator cannot reach InventoryService. It does not know if the request succeeded, failed, or was never received.

**Solution:**
- **Timeout with retry:** The orchestrator retries the inventory call with an idempotency key. If the service already processed it, the idempotent response is returned.
- **Saga timeout:** If retries exhaust, the orchestrator initiates compensation. Because earlier steps were recorded, compensations can proceed even if the saga picks up on a different orchestrator instance after failover.
- **Idempotency on compensation:** The refund call also uses an idempotency key, so retrying the compensation is safe.

### Scenario 3: Orchestrator Crashes Mid-Saga

**Problem:** The orchestrator crashes after step 2 (payment charged) but before step 3 (inventory reserve).

**Solution:**
- The saga execution record is persisted to a durable store after each step
- On restart (or failover to another instance), the orchestrator reads the saga record
- It sees the saga is in `Running { completed_steps: ["create_order", "charge_payment"] }`
- It resumes from step 3

This is why saga state persistence is non-negotiable in production systems.

### Scenario 4: Duplicate Compensation

**Problem:** The orchestrator sends a refund, crashes before recording that the compensation completed, restarts, and sends the refund again.

**Solution:** Compensations must be idempotent. The `PaymentService.refund()` endpoint uses an idempotency key tied to the saga ID and step name. The second refund request returns the same result without double-refunding.

---

## Choreography vs Orchestration Decision Guide

| Factor | Choreography | Orchestration |
|---|---|---|
| Number of steps | 2-3 steps | 4+ steps |
| Team ownership | Each service team owns their part | Central team owns the flow |
| Observability needs | Low (simple flows) | High (complex flows, SLA tracking) |
| Compensation complexity | Simple, symmetric compensations | Complex compensations with ordering |
| Coupling tolerance | Low coupling preferred | Some coupling acceptable |
| Debugging ease | Harder (event traces needed) | Easier (orchestrator logs) |

---

## Common Pitfalls

1. **Skipping compensation design:** Teams often build the happy path and defer compensations to "later." Later never comes. Design compensations alongside forward actions.

2. **Non-idempotent compensations:** If a compensation is retried and it is not idempotent, you get double refunds, double cancellations, or corrupted state. Every compensation must be idempotent.

3. **Ignoring compensation failure:** When a compensation fails, the system is in an inconsistent state. This must trigger an alert for manual intervention, not a silent log line.

4. **Using 2PC across microservices:** 2PC requires all participants to be available and responsive. In a microservice architecture with independent deployment cycles and varying availability, 2PC creates brittle coupling. Use sagas instead.

5. **Mixing saga and non-saga transactions:** If some steps use sagas and others use local transactions that assume atomicity with external services, the overall consistency guarantee is undefined.

---

## Key Takeaways

- 2PC provides atomicity but at the cost of availability, latency, and tight coupling. Reserve it for within-database or within-datacenter scenarios where a coordinator can be made highly available (e.g., Spanner's Paxos-backed 2PC).
- Sagas are the standard pattern for cross-service data consistency in microservice architectures. They trade atomicity for availability and loose coupling.
- Orchestration is preferable for complex flows (4+ steps, complex compensations). Choreography works well for simple, loosely coupled flows.
- Every compensation must be idempotent and must handle its own failure gracefully.
- Saga state must be persisted durably. Without persistence, a crash mid-saga leaves the system in an unrecoverable inconsistent state.
- The combination of sagas + idempotency keys + persistent state machines provides a practical approach to data consistency that scales across services and failure modes.
