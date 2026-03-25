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

```text
STRUCTURE StoredEvent:
    aggregate_id ← UUID
    aggregate_type ← string
    event_type ← string
    event_data ← JSON value
    metadata ← optional JSON value
    version ← integer

STRUCTURE EventStore:
    pool ← PgPool

/// Append new events for an aggregate, enforcing expected version.
/// Returns an error if another writer has appended events since we last read.
PROCEDURE APPEND(store, aggregate_id, aggregate_type, expected_version, events):
    tx ← AWAIT BEGIN_TRANSACTION(store.pool)

    FOR EACH (i, (event_type, event_data)) IN ENUMERATE(events):
        version ← expected_version + 1 + i
        AWAIT EXECUTE tx:
            "INSERT INTO events (aggregate_id, aggregate_type, event_type, event_data, version)
             VALUES (aggregate_id, aggregate_type, event_type, event_data, version)"
        IF unique constraint violation THEN
            // A concurrent write happened
            RETURN Error(ConcurrencyConflict)

    AWAIT COMMIT(tx)

/// Load all events for an aggregate, ordered by version.
PROCEDURE LOAD(store, aggregate_id):
    events ← AWAIT QUERY store.pool:
        "SELECT aggregate_id, aggregate_type, event_type, event_data, metadata, version
         FROM events WHERE aggregate_id = aggregate_id ORDER BY version"
    RETURN events

/// Load events after a specific version (used with snapshots).
PROCEDURE LOAD_AFTER_VERSION(store, aggregate_id, after_version):
    events ← AWAIT QUERY store.pool:
        "SELECT aggregate_id, aggregate_type, event_type, event_data, metadata, version
         FROM events WHERE aggregate_id = aggregate_id AND version > after_version
         ORDER BY version"
    RETURN events
```

## Rebuilding Aggregates from Events

The aggregate is the domain object whose state is derived entirely from its event history. Commands validate business rules against the current state and produce new events.

```text
ENUMERATION BankAccountEvent:
    Opened { account_id, owner, initial_balance }
    Deposited { amount, source }
    Withdrawn { amount, reason }
    Frozen { reason }

STRUCTURE BankAccount:
    id ← UUID
    owner ← string
    balance ← Money
    is_frozen ← boolean
    version ← integer  // tracks the last applied event version

/// Rebuild state by replaying events from the store.
PROCEDURE FROM_EVENTS(events):
    account ← None

    FOR EACH stored IN events:
        event ← PARSE_JSON(stored.event_data) AS BankAccountEvent

        MATCH event:
            Opened { account_id, owner, initial_balance } →
                account ← BankAccount {
                    id ← account_id, owner ← owner,
                    balance ← initial_balance, is_frozen ← false,
                    version ← stored.version
                }
            Deposited { amount, ... } →
                IF account IS present THEN
                    account.balance ← account.balance + amount
                    account.version ← stored.version
            Withdrawn { amount, ... } →
                IF account IS present THEN
                    account.balance ← account.balance - amount
                    account.version ← stored.version
            Frozen { ... } →
                IF account IS present THEN
                    account.is_frozen ← true
                    account.version ← stored.version

    RETURN account

/// Command: validates business rules, produces events on success.
PROCEDURE WITHDRAW(account, amount, reason):
    IF account.is_frozen THEN
        RETURN Error(AccountFrozen)
    IF amount > account.balance THEN
        RETURN Error(InsufficientFunds)
    RETURN Withdrawn { amount, reason }

PROCEDURE DEPOSIT(account, amount, source):
    IF account.is_frozen THEN
        RETURN Error(AccountFrozen)
    RETURN Deposited { amount, source }
```

### The Command-Event Cycle

The full cycle for processing a command:

1. Load events for the aggregate from the event store.
2. Replay events to rebuild the current state.
3. Execute the command against the current state (validate business rules).
4. If valid, append the resulting events to the store with the expected version.
5. If a concurrency conflict occurs, reload and retry.

```text
PROCEDURE HANDLE_WITHDRAWAL(store, account_id, amount, reason):
    events ← AWAIT LOAD(store, account_id)
    account ← FROM_EVENTS(events)
    IF account IS None THEN RETURN Error(NotFound)

    new_event ← WITHDRAW(account, amount, reason)
    IF new_event IS error THEN RETURN error
    event_data ← TO_JSON(new_event)

    AWAIT APPEND(store, account_id, "BankAccount", account.version,
        [("Withdrawn", event_data)])
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
