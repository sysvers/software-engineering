# Event Sourcing

## The Core Idea

Instead of storing the current state of an entity, event sourcing stores the sequence of events that led to the current state. The current state is derived by replaying events.

**Traditional (state-based):**
```
Database row: { id: 1, balance: 750, name: "Alice" }
```
You know the current balance, but not *how* it got there.

**Event sourced:**
```
Event 1: AccountOpened { id: 1, name: "Alice", initial_deposit: 1000 }
Event 2: MoneyWithdrawn { id: 1, amount: 200, reason: "ATM" }
Event 3: MoneyDeposited { id: 1, amount: 50, source: "transfer" }
Event 4: MoneyWithdrawn { id: 1, amount: 100, reason: "purchase" }

Current state = replay all events -> balance: 750
```

You have a complete audit trail: every change, when it happened, and why. You can answer questions like "What was the balance on Tuesday?" by replaying events up to that point.

## The Event Store

An event store is an append-only database optimized for storing and retrieving events. Events are never updated or deleted.

### Schema

```sql
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    event_type      TEXT NOT NULL,
    event_data      JSONB NOT NULL,
    metadata        JSONB,          -- correlation_id, user_id, etc.
    version         INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE (aggregate_id, version)  -- Optimistic concurrency control
);

CREATE INDEX idx_events_aggregate ON events (aggregate_id, version);
```

### Optimistic Concurrency

The `(aggregate_id, version)` unique constraint prevents two concurrent writes from creating conflicting events. If two processes try to write version 5 for the same aggregate, one will fail with a unique constraint violation -- telling it to reload and retry.

This is the event sourcing equivalent of optimistic locking. No row-level locks are needed. The constraint enforces a linear event history per aggregate.

### Rust Event Store Implementation

```rust
use serde::{Serialize, Deserialize};
use sqlx::PgPool;
use uuid::Uuid;

#[derive(Clone, Debug)]
struct StoredEvent {
    aggregate_id: Uuid,
    aggregate_type: String,
    event_type: String,
    event_data: serde_json::Value,
    metadata: Option<serde_json::Value>,
    version: i32,
}

struct EventStore {
    pool: PgPool,
}

impl EventStore {
    /// Append new events for an aggregate, enforcing expected version.
    /// Returns an error if another writer has appended events since we last read.
    async fn append(
        &self,
        aggregate_id: Uuid,
        aggregate_type: &str,
        expected_version: i32,
        events: Vec<(String, serde_json::Value)>,
    ) -> Result<(), EventStoreError> {
        let mut tx = self.pool.begin().await?;

        for (i, (event_type, event_data)) in events.into_iter().enumerate() {
            let version = expected_version + 1 + i as i32;
            sqlx::query(
                "INSERT INTO events (aggregate_id, aggregate_type, event_type, event_data, version)
                 VALUES ($1, $2, $3, $4, $5)"
            )
            .bind(aggregate_id)
            .bind(aggregate_type)
            .bind(&event_type)
            .bind(&event_data)
            .bind(version)
            .execute(&mut *tx)
            .await
            .map_err(|e| {
                // Unique constraint violation means a concurrent write happened
                if is_unique_violation(&e) {
                    EventStoreError::ConcurrencyConflict
                } else {
                    EventStoreError::Database(e)
                }
            })?;
        }

        tx.commit().await?;
        Ok(())
    }

    /// Load all events for an aggregate, ordered by version.
    async fn load(&self, aggregate_id: Uuid) -> Result<Vec<StoredEvent>, EventStoreError> {
        let events = sqlx::query_as!(
            StoredEvent,
            "SELECT aggregate_id, aggregate_type, event_type, event_data, metadata, version
             FROM events WHERE aggregate_id = $1 ORDER BY version",
            aggregate_id
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(events)
    }

    /// Load events after a specific version (used with snapshots).
    async fn load_after_version(
        &self,
        aggregate_id: Uuid,
        after_version: i32,
    ) -> Result<Vec<StoredEvent>, EventStoreError> {
        let events = sqlx::query_as!(
            StoredEvent,
            "SELECT aggregate_id, aggregate_type, event_type, event_data, metadata, version
             FROM events WHERE aggregate_id = $1 AND version > $2 ORDER BY version",
            aggregate_id, after_version
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(events)
    }
}
```

## Rebuilding Aggregates from Events

The aggregate is the domain object whose state is derived entirely from its event history. Commands validate business rules against the current state and produce new events.

```rust
#[derive(Clone, Serialize, Deserialize)]
enum BankAccountEvent {
    Opened { account_id: Uuid, owner: String, initial_balance: Money },
    Deposited { amount: Money, source: String },
    Withdrawn { amount: Money, reason: String },
    Frozen { reason: String },
}

struct BankAccount {
    id: Uuid,
    owner: String,
    balance: Money,
    is_frozen: bool,
    version: i32,  // tracks the last applied event version
}

impl BankAccount {
    /// Rebuild state by replaying events from the store.
    fn from_events(events: &[StoredEvent]) -> Option<Self> {
        let mut account: Option<Self> = None;

        for stored in events {
            let event: BankAccountEvent =
                serde_json::from_value(stored.event_data.clone()).ok()?;

            match event {
                BankAccountEvent::Opened { account_id, owner, initial_balance } => {
                    account = Some(BankAccount {
                        id: account_id,
                        owner,
                        balance: initial_balance,
                        is_frozen: false,
                        version: stored.version,
                    });
                }
                BankAccountEvent::Deposited { amount, .. } => {
                    if let Some(ref mut acc) = account {
                        acc.balance += amount;
                        acc.version = stored.version;
                    }
                }
                BankAccountEvent::Withdrawn { amount, .. } => {
                    if let Some(ref mut acc) = account {
                        acc.balance -= amount;
                        acc.version = stored.version;
                    }
                }
                BankAccountEvent::Frozen { .. } => {
                    if let Some(ref mut acc) = account {
                        acc.is_frozen = true;
                        acc.version = stored.version;
                    }
                }
            }
        }

        account
    }

    /// Command: validates business rules, produces events on success.
    fn withdraw(&self, amount: Money, reason: String) -> Result<BankAccountEvent, AccountError> {
        if self.is_frozen {
            return Err(AccountError::AccountFrozen);
        }
        if amount > self.balance {
            return Err(AccountError::InsufficientFunds);
        }
        Ok(BankAccountEvent::Withdrawn { amount, reason })
    }

    fn deposit(&self, amount: Money, source: String) -> Result<BankAccountEvent, AccountError> {
        if self.is_frozen {
            return Err(AccountError::AccountFrozen);
        }
        Ok(BankAccountEvent::Deposited { amount, source })
    }
}
```

### The Command-Event Cycle

The full cycle for processing a command:

1. Load events for the aggregate from the event store.
2. Replay events to rebuild the current state.
3. Execute the command against the current state (validate business rules).
4. If valid, append the resulting events to the store with the expected version.
5. If a concurrency conflict occurs, reload and retry.

```rust
async fn handle_withdrawal(
    store: &EventStore,
    account_id: Uuid,
    amount: Money,
    reason: String,
) -> Result<(), AccountError> {
    let events = store.load(account_id).await?;
    let account = BankAccount::from_events(&events)
        .ok_or(AccountError::NotFound)?;

    let new_event = account.withdraw(amount, reason)?;
    let event_data = serde_json::to_value(&new_event)?;

    store.append(
        account_id,
        "BankAccount",
        account.version,
        vec![("Withdrawn".into(), event_data)],
    ).await?;

    Ok(())
}
```

## Real-World Examples

### LMAX Exchange

LMAX built a foreign exchange trading platform using event sourcing. All market events are stored in sequence. The trading engine replays events to rebuild state after a restart. This architecture enables them to process 6 million transactions per second on a single thread -- because the event log is the system of record, and the trading engine is a pure function of events.

The single-threaded design eliminates lock contention entirely. The event journal on disk is the source of truth. In-memory state is a cache that can be rebuilt at any time. On restart, LMAX replays the journal and resumes trading within seconds.

### Banking and Financial Systems

Banks are natural fits for event sourcing. Every transaction (deposit, withdrawal, transfer) is an event. The account balance is a projection. This is not new -- double-entry bookkeeping (invented in the 15th century) is essentially event sourcing. Modern banks like ING and Capital One use event sourcing for their core banking platforms, where the audit trail is not a feature -- it is a regulatory requirement.

## When to Use Event Sourcing

**Use for:**
- Financial systems (audit trail is essential)
- Systems where "why did this state change?" needs an answer
- Domains with complex state transitions
- When you need temporal queries ("what was the state on date X?")

**Avoid for:**
- Simple CRUD applications
- Systems with simple, rarely-changing state
- Teams without experience in event-driven systems (high learning curve)
- Data subject to GDPR deletion requirements (events are immutable, so deletion requires special handling like crypto-shredding)

## Key Takeaways

1. Event sourcing stores the history, not just the current state. Current state is derived by replaying events.
2. The event store is append-only. Events are immutable facts.
3. Optimistic concurrency via version numbers prevents conflicting writes without locks.
4. Aggregates are rebuilt from their event history. Commands validate rules and produce new events.
5. Event sourcing gives you a complete audit trail and temporal queries for free, but adds complexity that must be justified by the domain.
