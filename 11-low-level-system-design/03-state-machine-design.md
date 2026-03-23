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

```rust
enum SubscriptionState {
    Trial { expires_at: DateTime<Utc> },
    Active { plan: Plan, renews_at: DateTime<Utc> },
    Expired { expired_at: DateTime<Utc> },
    Cancelled { cancelled_at: DateTime<Utc>, reason: String },
}

impl SubscriptionState {
    fn activate(self, plan: Plan) -> Result<Self, SubscriptionError> {
        match self {
            Self::Trial { .. } => Ok(Self::Active {
                plan,
                renews_at: Utc::now() + Duration::days(30),
            }),
            _ => Err(SubscriptionError::InvalidTransition(
                "Can only activate from trial"
            )),
        }
    }

    fn cancel(self, reason: String) -> Result<Self, SubscriptionError> {
        match self {
            Self::Trial { .. } | Self::Active { .. } => Ok(Self::Cancelled {
                cancelled_at: Utc::now(),
                reason,
            }),
            _ => Err(SubscriptionError::InvalidTransition(
                "Cannot cancel expired or already cancelled subscription"
            )),
        }
    }

    fn expire(self) -> Result<Self, SubscriptionError> {
        match self {
            Self::Trial { .. } | Self::Active { .. } => Ok(Self::Expired {
                expired_at: Utc::now(),
            }),
            _ => Err(SubscriptionError::InvalidTransition(
                "Already expired or cancelled"
            )),
        }
    }
}
```

**Key design choice:** Transition methods consume `self` (take ownership). This means the old state no longer exists after a transition — you cannot accidentally use the old state. The caller must use the returned new state.

### Example: Order Lifecycle

```rust
enum OrderState {
    Draft { items: Vec<LineItem> },
    Submitted { items: Vec<LineItem>, submitted_at: DateTime<Utc> },
    Paid { items: Vec<LineItem>, payment_id: PaymentId, paid_at: DateTime<Utc> },
    Shipped { tracking_number: String, shipped_at: DateTime<Utc> },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String, cancelled_at: DateTime<Utc> },
}

impl OrderState {
    fn submit(self) -> Result<Self, OrderError> {
        match self {
            Self::Draft { items } if !items.is_empty() => Ok(Self::Submitted {
                items,
                submitted_at: Utc::now(),
            }),
            Self::Draft { items } if items.is_empty() => {
                Err(OrderError::EmptyOrder)
            }
            _ => Err(OrderError::InvalidTransition("Can only submit from draft")),
        }
    }

    fn pay(self, payment_id: PaymentId) -> Result<Self, OrderError> {
        match self {
            Self::Submitted { items, .. } => Ok(Self::Paid {
                items,
                payment_id,
                paid_at: Utc::now(),
            }),
            _ => Err(OrderError::InvalidTransition("Can only pay a submitted order")),
        }
    }

    fn ship(self, tracking_number: String) -> Result<Self, OrderError> {
        match self {
            Self::Paid { .. } => Ok(Self::Shipped {
                tracking_number,
                shipped_at: Utc::now(),
            }),
            _ => Err(OrderError::InvalidTransition("Can only ship a paid order")),
        }
    }
}
```

## The Typestate Pattern

The typestate pattern goes further: it makes invalid transitions a **compile-time error** rather than a runtime error. Each state is a separate type, and methods only exist on valid source states.

```rust
struct Connection<S: ConnectionState> {
    addr: SocketAddr,
    _state: PhantomData<S>,
}

// State types — zero-sized, exist only for the type system
struct Disconnected;
struct Connecting;
struct Connected;

// Trait to mark valid states
trait ConnectionState {}
impl ConnectionState for Disconnected {}
impl ConnectionState for Connecting {}
impl ConnectionState for Connected {}

// Methods only available in specific states
impl Connection<Disconnected> {
    fn connect(self) -> Connection<Connecting> {
        // Start connection attempt
        Connection { addr: self.addr, _state: PhantomData }
    }
}

impl Connection<Connecting> {
    fn on_connected(self) -> Connection<Connected> {
        Connection { addr: self.addr, _state: PhantomData }
    }

    fn on_failed(self) -> Connection<Disconnected> {
        Connection { addr: self.addr, _state: PhantomData }
    }
}

impl Connection<Connected> {
    fn send(&self, data: &[u8]) -> Result<(), SendError> {
        // Can only send when connected — enforced by the type system
        todo!()
    }

    fn disconnect(self) -> Connection<Disconnected> {
        Connection { addr: self.addr, _state: PhantomData }
    }
}
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

```rust
// Store as a tagged enum in the database
#[derive(Serialize, Deserialize)]
#[serde(tag = "state")]
enum SubscriptionState {
    Trial { expires_at: DateTime<Utc> },
    Active { plan: Plan, renews_at: DateTime<Utc> },
    Expired { expired_at: DateTime<Utc> },
    Cancelled { cancelled_at: DateTime<Utc>, reason: String },
}

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
