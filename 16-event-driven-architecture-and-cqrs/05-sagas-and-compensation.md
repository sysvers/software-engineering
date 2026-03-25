# Sagas and Compensation

## The Problem: Distributed Transactions

In a monolith, you wrap multiple operations in a single database transaction. If anything fails, everything rolls back. In a distributed system with multiple services, each owning its own database, there is no single transaction boundary.

```
Monolith:
  BEGIN TRANSACTION
    reserve_inventory()
    charge_payment()
    create_shipment()
  COMMIT  -- all or nothing

Microservices:
  InventoryService (owns its DB) -- reserve stock
  PaymentService   (owns its DB) -- charge card
  ShippingService  (owns its DB) -- create shipment

  No shared transaction. What happens when payment succeeds but shipping fails?
```

Two-phase commit (2PC) is the traditional solution for distributed transactions. A coordinator asks all participants to prepare, then tells them to commit. If any participant cannot prepare, all abort.

```
Coordinator:
  Phase 1 (Prepare):  "Can you commit?"
    InventoryService -> "Yes, prepared"
    PaymentService   -> "Yes, prepared"
    ShippingService  -> "Yes, prepared"

  Phase 2 (Commit):   "Commit now"
    All services commit.

  If any says "No" in Phase 1:
    "Abort" -> All services roll back.
```

**Why 2PC is rarely used across services:**

- **Blocking.** All participants hold locks during both phases. If the coordinator crashes between Phase 1 and Phase 2, participants are stuck holding locks indefinitely.
- **Availability.** Every participant must be reachable. One service being down blocks the entire transaction.
- **Latency.** Two network round-trips minimum. This does not scale to high-throughput systems.
- **Coupling.** All services must support the same 2PC protocol, typically through a shared transaction manager like XA.

2PC works well within a single database cluster. It works poorly across independent services with different databases, different teams, and different deployment schedules. The saga pattern is the practical alternative.

## The Saga Pattern

A saga replaces a single distributed transaction with a sequence of local transactions. Each local transaction updates one service and publishes an event or sends a command to trigger the next step. If a step fails, previously completed steps are undone through *compensating actions*.

```
Forward actions:               Compensating actions:
1. Reserve inventory           -> Release inventory
2. Charge payment              -> Refund payment
3. Create shipment             -> Cancel shipment

If step 2 (charge payment) fails:
  -> Compensate step 1: release inventory
  -> Report failure to user

If step 3 (create shipment) fails:
  -> Compensate step 2: refund payment
  -> Compensate step 1: release inventory
  -> Report failure to user
```

**Critical rule:** Every forward action must have a compensating action. Compensation is not the same as "undo" -- you cannot un-send an email or un-charge a credit card. Compensation is a *semantic reverse*: you send a correction email, you issue a refund.

## Choreography vs Orchestration

There are two ways to coordinate a saga's steps.

### Choreography

Each service listens for events and decides what to do next. There is no central coordinator.

```
OrderService publishes OrderPlaced
  -> InventoryService hears it, reserves stock, publishes StockReserved
    -> PaymentService hears it, charges card, publishes PaymentCompleted
      -> ShippingService hears it, creates shipment, publishes ShipmentCreated
        -> OrderService hears it, marks order as complete

If PaymentService fails:
  PaymentService publishes PaymentFailed
    -> InventoryService hears it, releases stock
    -> OrderService hears it, marks order as failed
```

**Advantages:**
- No single point of coordination. Each service is autonomous.
- Easy to add new consumers without modifying existing services.
- Naturally aligns with event-driven architecture.

**Disadvantages:**
- The business flow is scattered across services. No single place shows the full workflow.
- Adding a new step requires understanding which services react to which events.
- Cycles and feedback loops can emerge, making the flow hard to reason about.
- Difficult to answer "what step is this order on?" without querying multiple services.

Choreography works well for simple flows with few steps (two or three services). Beyond that, the implicit flow becomes hard to maintain.

### Orchestration

A central saga orchestrator explicitly directs each step. It sends commands to services and reacts to their responses.

```
OrderSaga (orchestrator):
  1. Send ReserveStock command to InventoryService
  2. Receive StockReserved -> Send ChargePayment to PaymentService
  3. Receive PaymentCompleted -> Send CreateShipment to ShippingService
  4. Receive ShipmentCreated -> Mark order as complete

  If any step fails:
    Run compensating actions in reverse order
```

**Advantages:**
- The entire business flow is visible in one place -- the orchestrator.
- Easy to add, remove, or reorder steps.
- Easy to answer "what step is this order on?" -- the orchestrator tracks it.
- Complex failure handling logic lives in one place.

**Disadvantages:**
- The orchestrator knows about all participating services (higher coupling).
- Risk of the orchestrator becoming a "god object" with too much logic.
- The orchestrator must be highly available -- if it goes down, in-flight sagas stall.

Orchestration is the better choice for complex, multi-step business processes where visibility and explicit failure handling matter.

## Rust Implementation: Saga Orchestrator

```text
/// Each step in a saga: a forward action and its compensating action.
STRUCTURE SagaStep:
    name ← string
    action ← FUNCTION(context) → Result
    compensation ← FUNCTION(context) → Result

ENUMERATION SagaError:
    StepFailed { step, reason }
    CompensationFailed { step, reason }

/// A generic saga orchestrator that runs steps in order and compensates on failure.
STRUCTURE SagaOrchestrator:
    steps ← list of SagaStep

PROCEDURE NEW_SAGA():
    RETURN SagaOrchestrator { steps ← empty list }

PROCEDURE ADD_STEP(saga, name, action, compensation):
    APPEND SagaStep { name, action, compensation } TO saga.steps

/// Execute the saga. On failure, compensate all completed steps in reverse order.
PROCEDURE EXECUTE(saga, context):
    completed ← empty list

    FOR EACH (i, step) IN ENUMERATE(saga.steps):
        PRINT "[saga] Executing step:", step.name
        result ← step.action(context)
        IF result IS Ok THEN
            APPEND i TO completed
        ELSE
            PRINT "[saga] Step '" + step.name + "' failed, compensating..."
            // Compensate in reverse order
            FOR EACH j IN REVERSE(completed):
                PRINT "[saga] Compensating step:", saga.steps[j].name
                comp_result ← saga.steps[j].compensation(context)
                IF comp_result IS error THEN
                    // Compensation failure is serious -- log and continue
                    LOG CRITICAL: comp_result
            RETURN result (error)

    PRINT "[saga] All steps completed successfully"
    RETURN Ok

// --- Application-specific context and usage ---

STRUCTURE OrderContext:
    order_id ← UUID
    customer_id ← string
    amount ← float
    inventory_reserved ← boolean
    payment_charged ← boolean
    shipment_created ← boolean

PROCEDURE BUILD_ORDER_SAGA():
    saga ← NEW_SAGA()

    ADD_STEP(saga, "reserve_inventory",
        action: FUNCTION(ctx):
            // In production: send command to InventoryService, await response
            PRINT "  Reserving inventory for order", ctx.order_id
            ctx.inventory_reserved ← true
            RETURN Ok,
        compensation: FUNCTION(ctx):
            PRINT "  Releasing inventory for order", ctx.order_id
            ctx.inventory_reserved ← false
            RETURN Ok
    )

    ADD_STEP(saga, "charge_payment",
        action: FUNCTION(ctx):
            PRINT "  Charging $" + ctx.amount + " for order", ctx.order_id
            // Simulate a failure for demonstration:
            // RETURN Error(StepFailed { step ← "charge_payment", reason ← "Card declined" })
            ctx.payment_charged ← true
            RETURN Ok,
        compensation: FUNCTION(ctx):
            PRINT "  Refunding $" + ctx.amount + " for order", ctx.order_id
            ctx.payment_charged ← false
            RETURN Ok
    )

    ADD_STEP(saga, "create_shipment",
        action: FUNCTION(ctx):
            PRINT "  Creating shipment for order", ctx.order_id
            ctx.shipment_created ← true
            RETURN Ok,
        compensation: FUNCTION(ctx):
            PRINT "  Cancelling shipment for order", ctx.order_id
            ctx.shipment_created ← false
            RETURN Ok
    )

    RETURN saga
```

The orchestrator is generic over `Ctx`, so it can be reused for different saga types. In a production system, the actions would send commands over a message broker and the orchestrator would be async, persisting its state between steps so it can survive restarts.

## Compensating Actions: Design Guidelines

Not every action has a trivial reverse. Designing compensating actions requires careful thought.

| Forward Action | Compensation | Notes |
|---------------|-------------|-------|
| Reserve inventory | Release inventory | Straightforward reversal |
| Charge credit card | Issue refund | Refund is a new transaction, not a rollback |
| Send email | Send correction email | Cannot un-send; can only follow up |
| Create database record | Mark as cancelled (soft delete) | Do not hard-delete; maintain audit trail |
| Publish event | Publish compensating event | Consumers must handle both |
| Call third-party API | Call cancellation endpoint | Not always possible; may require manual intervention |

**Key principles:**

1. **Compensations must be idempotent.** A compensation might be retried if the first attempt's result is unknown. Refunding twice is worse than not refunding.
2. **Compensations can fail.** Have a strategy: retry with backoff, dead-letter queue for manual intervention, or alerting.
3. **Some actions are not compensatable.** Sending a physical package, triggering a bank wire, or calling an external API with no cancellation endpoint. Design the saga so non-compensatable steps run last.

## Failure Scenarios

### Happy Path

All steps succeed in order. No compensation needed. The saga completes and the order moves to "confirmed."

### Mid-Saga Failure

Payment fails after inventory is reserved. The orchestrator runs compensations for all completed steps in reverse: release inventory. The order is marked as "failed" with a reason.

### Compensation Failure

The most dangerous scenario. A forward step fails, and then a compensating action also fails. For example, payment fails and the inventory release call times out.

Strategies:
- **Retry with exponential backoff.** Most transient failures resolve on retry.
- **Dead-letter queue.** After N retries, park the failed compensation for manual review.
- **Alerting.** Compensation failures need immediate human attention -- the system is in an inconsistent state.
- **Reconciliation jobs.** Periodic background processes that detect and fix inconsistencies.

### Orchestrator Crash

If the orchestrator crashes mid-saga, it must resume when it restarts. This requires persisting the saga state (current step, completed steps, context) to a durable store. On restart, load incomplete sagas and continue from where they left off.

## Real-World Examples

### E-Commerce Order Flow

A customer places an order. The saga coordinates:

```
1. Validate order         -> No compensation needed (read-only)
2. Reserve inventory      -> Release inventory
3. Calculate tax          -> No compensation needed (read-only)
4. Authorize payment      -> Void authorization
5. Confirm order          -> Cancel order
6. Send confirmation      -> (non-compensatable, runs last)
```

Notice: read-only steps (validate, calculate tax) do not need compensation. The non-compensatable step (send confirmation) is placed last, so it only runs after all compensatable steps have succeeded.

### Banking Transfer

Transfer $500 from Account A to Account B.

```
1. Debit Account A ($500)   -> Credit Account A ($500)
2. Credit Account B ($500)  -> Debit Account B ($500)
```

If step 2 fails (Account B is closed), the compensation credits the $500 back to Account A. In practice, banks use a held/pending state: step 1 places a hold on $500 rather than debiting immediately, and the hold is either finalized or released based on step 2's outcome.

### Travel Booking

Book a flight, hotel, and car rental as a package.

```
1. Reserve flight seat     -> Cancel flight reservation
2. Reserve hotel room      -> Cancel hotel reservation
3. Reserve rental car      -> Cancel car reservation
4. Charge payment          -> Refund payment
```

If the hotel is fully booked (step 2 fails), the flight reservation from step 1 is cancelled. The customer is not charged because step 4 has not run yet. This is why the payment step comes after all reservations -- it minimizes refunds.

## Choreography vs Orchestration: When to Use Each

| Factor | Choreography | Orchestration |
|--------|-------------|---------------|
| Number of steps | 2-3 | 4+ |
| Flow complexity | Linear, no branching | Branching, conditional steps |
| Visibility | Distributed across services | Centralized in orchestrator |
| Team structure | Independent teams, loose coordination | Team owns the business process end-to-end |
| Failure handling | Each service handles its own | Centralized failure and compensation logic |
| Adding new steps | Add a new consumer | Modify the orchestrator |
| Debugging | Requires distributed tracing | Read the orchestrator's state |

In practice, many systems use a hybrid: choreography for simple, loosely-coupled event reactions (send email when user registers) and orchestration for complex multi-step business processes (order fulfillment, payment processing).

## Key Takeaways

1. Distributed transactions (2PC) block participants and reduce availability. They work within a database cluster but not across independent services.
2. The saga pattern replaces one distributed transaction with a sequence of local transactions plus compensating actions.
3. Every forward action must have a defined compensating action. Design for the fact that compensation can also fail.
4. Choreography distributes the flow across services -- good for simple flows, hard to debug for complex ones.
5. Orchestration centralizes the flow in one place -- easier to reason about, but the orchestrator becomes a coordination point.
6. Place non-compensatable steps (sending emails, calling irrevocable APIs) at the end of the saga.
7. Persist saga state so the orchestrator can resume after a crash. In-memory-only sagas will lose in-flight work.
