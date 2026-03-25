# State Machine Design

Many systems have well-defined states and transitions. Modeling them explicitly prevents invalid states and makes the system predictable. If your entity has defined states and transitions, encode them in the type system rather than scattering transition logic across the codebase.

## Why Explicit State Machines?

Without explicit state modeling, state transitions are scattered across the codebase as if-else chains. Different parts of the code may have different assumptions about which transitions are valid. Bugs arise when code attempts an invalid transition that was never tested.

With explicit state machines:
- Valid transitions are defined in one place.
- Invalid transitions are rejected at compile time (typestate) or runtime (enum-based).
- Every state carries exactly the data relevant to that state.

## Enum-Based State Machines

The simplest approach: use a Rust enum where each variant represents a state, with transition methods that consume `self` and return the new state.

### Example: Subscription Lifecycle

```
                    ┌──────────────┐
                    │              │
         ┌─────────▼──┐      ┌───┴────────┐
    ──→  │   Trial    │──→   │   Active   │
         └─────┬──────┘      └───┬────┬───┘
               │                 │    │
               │          ┌─────▼─┐  │
               └─────────→│ Expired│  │
                          └───────┘  │
                                     │
                          ┌──────────▼──┐
                          │  Cancelled  │
                          └─────────────┘
```

```text
ENUM SubscriptionState
    Trial { expires_at: DateTime }
    Active { plan: Plan, renews_at: DateTime }
    Expired { expired_at: DateTime }
    Cancelled { cancelled_at: DateTime, reason: String }

FUNCTION ACTIVATE(state: SubscriptionState, plan: Plan) → Result<SubscriptionState, SubscriptionError>
    MATCH state
        CASE Trial { .. }:
            RETURN Active { plan ← plan, renews_at ← NOW() + 30 days }
        DEFAULT:
            RETURN Error("Can only activate from trial")

FUNCTION CANCEL(state: SubscriptionState, reason: String) → Result<SubscriptionState, SubscriptionError>
    MATCH state
        CASE Trial { .. } OR Active { .. }:
            RETURN Cancelled { cancelled_at ← NOW(), reason ← reason }
        DEFAULT:
            RETURN Error("Cannot cancel expired or already cancelled subscription")

FUNCTION EXPIRE(state: SubscriptionState) → Result<SubscriptionState, SubscriptionError>
    MATCH state
        CASE Trial { .. } OR Active { .. }:
            RETURN Expired { expired_at ← NOW() }
        DEFAULT:
            RETURN Error("Already expired or cancelled")
```

**Key design choice:** Transition methods consume `self` (take ownership). This means the old state no longer exists after a transition — you cannot accidentally use the old state. The caller must use the returned new state.

### Example: Order Lifecycle

```text
ENUM OrderState
    Draft { items: List<LineItem> }
    Submitted { items: List<LineItem>, submitted_at: DateTime }
    Paid { items: List<LineItem>, payment_id: PaymentId, paid_at: DateTime }
    Shipped { tracking_number: String, shipped_at: DateTime }
    Delivered { delivered_at: DateTime }
    Cancelled { reason: String, cancelled_at: DateTime }

FUNCTION SUBMIT(state: OrderState) → Result<OrderState, OrderError>
    MATCH state
        CASE Draft { items } WHERE items IS NOT EMPTY:
            RETURN Submitted { items ← items, submitted_at ← NOW() }
        CASE Draft { items } WHERE items IS EMPTY:
            RETURN Error(EmptyOrder)
        DEFAULT:
            RETURN Error("Can only submit from draft")

FUNCTION PAY(state: OrderState, payment_id: PaymentId) → Result<OrderState, OrderError>
    MATCH state
        CASE Submitted { items, .. }:
            RETURN Paid { items ← items, payment_id ← payment_id, paid_at ← NOW() }
        DEFAULT:
            RETURN Error("Can only pay a submitted order")

FUNCTION SHIP(state: OrderState, tracking_number: String) → Result<OrderState, OrderError>
    MATCH state
        CASE Paid { .. }:
            RETURN Shipped { tracking_number ← tracking_number, shipped_at ← NOW() }
        DEFAULT:
            RETURN Error("Can only ship a paid order")
```

## The Typestate Pattern

The typestate pattern goes further: it makes invalid transitions a **compile-time error** rather than a runtime error. Each state is a separate type, and methods only exist on valid source states.

```text
RECORD Connection<S: ConnectionState>
    addr: SocketAddr

// State types -- zero-sized, exist only for the type system
STATE TYPE Disconnected
STATE TYPE Connecting
STATE TYPE Connected

// Methods only available in specific states
FUNCTION CONNECT(conn: Connection<Disconnected>) → Connection<Connecting>
    // Start connection attempt
    RETURN Connection<Connecting> { addr ← conn.addr }

FUNCTION ON_CONNECTED(conn: Connection<Connecting>) → Connection<Connected>
    RETURN Connection<Connected> { addr ← conn.addr }

FUNCTION ON_FAILED(conn: Connection<Connecting>) → Connection<Disconnected>
    RETURN Connection<Disconnected> { addr ← conn.addr }

FUNCTION SEND(conn: Connection<Connected>, data: Bytes) → Result<Void, SendError>
    // Can only send when connected -- enforced by the type system
    // ...

FUNCTION DISCONNECT(conn: Connection<Connected>) → Connection<Disconnected>
    RETURN Connection<Disconnected> { addr ← conn.addr }
```

With this design, calling `.send()` on a `Connection<Disconnected>` is a compile error. You cannot forget to check the connection state.

**Trade-off:** Typestate is powerful but adds complexity. Use it for critical state machines (connections, security contexts, resource lifecycles) where an invalid transition would cause serious bugs. For simpler cases, enum-based state machines with runtime checks are sufficient.

## Real-World Example: Discord Voice Connections

Discord models voice connections as explicit state machines. A voice connection moves through states: Disconnected, Connecting, Connected, Reconnecting, Disconnected. Invalid transitions are impossible by design.

This approach eliminated an entire class of bugs where voice connections would enter invalid states, causing dropped calls. Before explicit state machines, race conditions between connection events (connect, disconnect, error) could leave the connection in a state that no code path expected.

Key lessons from Discord's approach:
- Every state transition is logged, making debugging straightforward.
- Reconnection logic is isolated to the `Reconnecting` state — it cannot interfere with other states.
- Timeouts are state-specific: a `Connecting` state has a timeout, but a `Connected` state does not.

## Persistence of State Machines

State machines need to survive process restarts. Serialize the current state to the database:

```text
// Store as a tagged enum in the database (serializable)
ENUM SubscriptionState [tagged by "state"]
    Trial { expires_at: DateTime }
    Active { plan: Plan, renews_at: DateTime }
    Expired { expired_at: DateTime }
    Cancelled { cancelled_at: DateTime, reason: String }

// In the database: {"state": "Active", "plan": "pro", "renews_at": "2026-04-23T00:00:00Z"}
```

Store an `audit_log` of transitions alongside the current state. This gives you a complete history of every state change, invaluable for debugging and compliance.

## When to Use Explicit State Machines

**Use them when:**
- The entity has more than two states.
- Transitions have preconditions or side effects.
- Invalid transitions would cause real bugs (dropped connections, incorrect billing, lost data).
- You need an audit trail of state changes.

**Skip them when:**
- The entity has two states (active/inactive) with trivial transitions.
- The state is a simple boolean flag.
- The added complexity is not justified by the risk of invalid states.
