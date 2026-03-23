# Architecture Decision Records (ADRs)

## Overview

An Architecture Decision Record is a short document that captures a significant architectural decision: what was decided, why, what alternatives were considered, and what consequences are expected. ADRs create institutional memory so that future team members (and future you) can understand not just what the system looks like, but why it looks that way.

The concept was popularized by Michael Nygard in a 2011 blog post and has since become standard practice at companies like ThoughtWorks, Spotify, and GitHub.

---

## Why ADRs Matter

Without ADRs, architectural knowledge lives in people's heads. When those people leave, go on vacation, or simply forget, the reasoning behind decisions is lost. Teams then face a painful choice: reverse-engineer the reasoning (expensive), blindly preserve the status quo (stagnant), or redo the decision from scratch (wasteful and risky).

ADRs solve this by making decisions searchable, reviewable, and auditable. They also force the decision-maker to articulate their reasoning, which often improves the quality of the decision itself.

---

## The ADR Template

Every ADR follows a consistent structure. The most widely used template has five sections:

```markdown
# ADR-NNN: [Short title describing the decision]

## Status
[Proposed | Accepted | Deprecated | Superseded by ADR-NNN]

## Context
What is the situation that motivates this decision? What forces are at play?
Include technical constraints, business requirements, team capabilities,
and timeline pressures. A reader who was not in the room should understand
the problem after reading this section.

## Decision
What is the change we are making? State it clearly and concisely.

## Alternatives Considered
What other options were evaluated? For each, briefly explain why it was
rejected. This is the most valuable section — it prevents future teams
from re-evaluating options that were already ruled out.

## Consequences
What are the expected results of this decision? Include both positive
and negative consequences. What trade-offs are we accepting? What new
constraints does this decision introduce?
```

Some teams add optional fields: **Date**, **Participants** (who was involved in the decision), and **Related ADRs** (links to prior or subsequent decisions).

---

## Sample ADRs

### ADR-001: Use an event-driven architecture for inter-module communication

**Status:** Accepted

**Context:**
Our modular monolith has five modules (Orders, Inventory, Payments, Shipping, Notifications). Currently, modules call each other directly through trait interfaces. As the system grows, we are seeing increasing coupling: changing the Inventory module's public API requires updating Orders and Shipping simultaneously. Deployment coordination is becoming a bottleneck.

**Decision:**
Adopt an in-process event bus for communication between modules. Modules publish domain events after completing operations. Other modules subscribe to events they care about. Direct trait-based calls remain available for queries that require an immediate response (e.g., checking inventory availability).

**Alternatives Considered:**
- **Keep direct calls only**: Simpler, but coupling will worsen as modules multiply. Every new cross-module interaction adds another trait dependency.
- **Extract to microservices with message broker**: Solves coupling but introduces distributed system complexity (network failures, eventual consistency, operational overhead). Our team of 8 engineers cannot absorb this cost yet.
- **Shared database views**: Modules read each other's data through database views. Rejected because it couples modules at the data layer, which is harder to evolve than API coupling.

**Consequences:**
- Modules are decoupled at the command/write path. Adding a new reaction to "OrderPlaced" requires no changes to the Orders module.
- Debugging becomes harder: event flow is implicit rather than visible in function call chains. We will add correlation IDs and structured logging to mitigate this.
- We accept eventual consistency for cross-module side effects (e.g., email confirmation may lag behind order placement by seconds).
- The event bus is an in-process implementation now, but can be replaced with Kafka/NATS later if we extract microservices.

---

### ADR-002: Use SQLx with compile-time query checking over an ORM

**Status:** Accepted

**Context:**
We need a strategy for database access in our Rust backend. The team has experience with ORMs (ActiveRecord, SQLAlchemy) from previous projects but is new to the Rust ecosystem. Our schema has 30+ tables with complex joins for reporting. We value type safety and want to catch query errors at compile time rather than at runtime.

**Decision:**
Use SQLx with compile-time verified queries for all database access. Write SQL directly rather than using an ORM query builder.

**Alternatives Considered:**
- **Diesel (Rust ORM)**: Strong type safety and compile-time checking. Rejected because its DSL becomes unwieldy for complex joins and reporting queries. The team found raw SQL more readable for our use case.
- **SeaORM**: More flexible than Diesel, supports async. Rejected because it adds an abstraction layer over SQL that obscures what queries actually execute, making performance tuning harder.
- **Raw sqlx without compile-time checking**: Faster compilation but loses the compile-time guarantee that queries match the schema. The safety benefit outweighs the compilation cost.

**Consequences:**
- All SQL queries are verified against the real database schema at compile time. Schema drift between code and database is caught before deployment.
- Developers must write raw SQL, which requires SQL proficiency. We will maintain a SQL style guide and include SQL reviews in PRs.
- Compile times increase because sqlx connects to the database during `cargo check`. We will use `SQLX_OFFLINE=true` with cached query metadata in CI to mitigate this.
- Migrations must be run before compiling. The development workflow requires a running PostgreSQL instance.

---

### ADR-003: Deprecate REST API in favor of gRPC for internal service communication

**Status:** Superseded by ADR-007

**Context:**
Our three internal services (Auth, Orders, Analytics) communicate over REST/JSON. Serialization overhead is significant for the Analytics service, which processes 50K events/second. We also lack a shared schema definition, leading to drift between client and server expectations. Two production incidents in the past quarter were caused by undocumented API changes.

**Decision:**
Adopt gRPC with Protocol Buffers for all internal service-to-service communication. REST remains for external-facing APIs consumed by third-party clients.

**Alternatives Considered:**
- **Keep REST with OpenAPI specs**: Adds schema documentation but does not solve serialization performance. OpenAPI code generation in Rust is less mature than protobuf.
- **GraphQL**: Flexible querying, but adds complexity for service-to-service calls where the query shape is known at compile time. Better suited for client-facing APIs.
- **Cap'n Proto / FlatBuffers**: Faster serialization than protobuf, but smaller ecosystems and less tooling in Rust. Protobuf's ecosystem maturity won.

**Consequences:**
- Shared `.proto` files serve as the source of truth for API contracts. Breaking changes are caught by the protobuf compiler.
- Binary serialization reduces payload size by approximately 60% compared to JSON for our Analytics payloads.
- The team must learn protobuf and gRPC. We will schedule a workshop and create internal documentation.
- Debugging is harder than REST (binary payloads are not human-readable). We will use grpcurl and Bloom RPC for manual testing.

---

## How to Write Good ADRs

**Keep them short.** An ADR should take 10-15 minutes to write and 5 minutes to read. If it exceeds two pages, the decision may be too large and should be split.

**Write the Context section for a stranger.** Assume the reader joined the team six months after the decision. They do not know the project history, the team dynamics, or the technical constraints. Spell it out.

**Be honest about trade-offs in Consequences.** Every decision has downsides. Listing only positive consequences suggests either a lack of analysis or an intent to sell rather than document. Future teams need to know the risks they inherited.

**Record the Alternatives section with genuine consideration.** Do not add straw-man alternatives that were never seriously evaluated. Each alternative should include the specific reason it was rejected, not just "we liked option A better."

**Use the decision to close debates.** Once accepted, an ADR settles the discussion. If circumstances change, write a new ADR that supersedes the old one. Do not edit accepted ADRs — they are a historical record.

---

## Tooling: adr-tools

The `adr-tools` package (available via Homebrew and most package managers) provides a CLI for managing ADRs:

```bash
# Initialize the ADR directory
adr init doc/architecture/decisions

# Create a new ADR
adr new "Use PostgreSQL as primary database"
# Creates: doc/architecture/decisions/0001-use-postgresql-as-primary-database.md

# Record that ADR 5 supersedes ADR 2
adr new -s 2 "Switch from REST to gRPC for internal APIs"

# List all ADRs
adr list

# Generate a table of contents
adr generate toc
```

For teams that prefer not to install tooling, a simple directory of numbered Markdown files works equally well. The value is in the content, not the tooling.

---

## Where to Store ADRs

**In the repository, next to the code.** ADRs should be version-controlled alongside the code they describe. The most common convention:

```
project-root/
├── doc/
│   └── architecture/
│       └── decisions/
│           ├── 0001-use-postgresql-as-primary-database.md
│           ├── 0002-use-sqlx-over-orm.md
│           ├── 0003-event-driven-inter-module-communication.md
│           └── README.md   # Table of contents
├── src/
└── ...
```

Some teams place ADRs in a `docs/adr/` directory at the repo root. The exact path matters less than consistency and discoverability.

**Do not store ADRs in a wiki.** Wikis drift from the code, lack version history tied to code changes, and are harder to find. An ADR in the repo is discoverable through `grep` and always in sync with the codebase version.

---

## Common Mistakes

### Writing ADRs after the fact
ADRs written weeks after a decision was made tend to omit the rejected alternatives and the real reasoning. Write the ADR as part of the decision process, not as documentation after the fact.

### Recording every decision
Not every decision needs an ADR. Reserve them for decisions that are architecturally significant: hard to reverse, affect multiple teams or modules, or constrain future choices. "Which JSON library to use" is probably not ADR-worthy. "Which database engine to use" almost certainly is.

### Never updating status
When a decision is superseded or deprecated, update the status field and link to the replacement ADR. A stale ADR with "Accepted" status that no longer reflects reality is worse than no ADR at all.

### Making ADRs too long
If an ADR reads like a research paper, it will not be read. Keep it to one page. If the decision requires extensive analysis, put the analysis in a separate design document and reference it from the ADR.

### Requiring unanimous approval
ADRs document decisions, not consensus. Record who participated and what the dissenting views were, but do not block the ADR on universal agreement. Disagreement is valuable context for future readers.

---

## Further Reading

- [Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) -- Michael Nygard's original blog post
- [adr-tools](https://github.com/npryce/adr-tools) -- CLI tooling for managing ADRs
- [ADR GitHub Organization](https://adr.github.io/) -- Templates, tooling, and examples
- *Design It!* -- Michael Keeling (2017) -- Covers ADRs as part of a broader architecture practice
- *Fundamentals of Software Architecture* -- Mark Richards & Neal Ford (2020) -- Chapter on documenting architecture decisions
