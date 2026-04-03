# System Design Ownership

At the Staff level, you own the design of systems that are too large or too important for a single team to own alone. This is not about drawing diagrams — it is about making the structural decisions that determine how fast your organization can move for the next two years.

---

## What System Design Ownership Means

Ownership at this level means you are the person who:

- Defines the boundaries between services and teams.
- Makes the trade-offs between consistency, availability, and performance.
- Decides when to build, buy, or defer.
- Ensures that the system as a whole is coherent, even when individual teams optimize locally.
- Is accountable when the design fails — not because you wrote the code, but because you set the direction.

## From Feature Design to System Design

Senior engineers design features. Staff engineers design systems. The difference:

| Feature Design | System Design |
|---------------|---------------|
| How do we implement search? | Where does search live in the architecture? |
| What is the API contract? | How do services discover and communicate? |
| How do we store this data? | How does data flow across the system? |
| What is the failure mode? | What is the blast radius of a failure? |

System design requires you to think in terms of boundaries, contracts, and failure domains rather than implementations.

## Key Responsibilities

### Defining Service Boundaries

The most consequential system design decision is where to draw the lines between services. Get it right, and teams can work independently. Get it wrong, and every feature requires cross-team coordination.

Good boundaries align with:
- **Business domains** — Each service maps to a business capability (orders, payments, inventory).
- **Team ownership** — One team owns one service. Shared ownership is no ownership.
- **Data ownership** — Each service owns its data. No shared databases.
- **Change frequency** — Things that change together should live together.

### Managing Technical Debt at Scale

Individual engineers accumulate debt within a codebase. Staff engineers see debt at the system level:

- A service that every other service depends on becomes a bottleneck.
- An inconsistent authentication pattern across services creates security risk.
- A messaging system chosen for simplicity five years ago cannot handle current scale.

Your job is to identify systemic debt, prioritize it against feature work, and create plans to address it incrementally. Nobody will give you a sprint for this. You need to weave it into ongoing work and make the case for dedicated investment when needed.

### Design Reviews

You should be reviewing the design of any system change that:
- Crosses service boundaries.
- Changes a public API contract.
- Introduces a new data store or messaging system.
- Affects more than one team's deploy pipeline.

A good design review is not a gate — it is a conversation. Your goal is to catch structural problems early, share context that the authoring team may not have, and ensure consistency across the system.

## Scaling Your Design Work

You cannot be in every design review for every team. Scale through:

- **Written standards.** Document the patterns and anti-patterns for your system. Teams should be able to make good decisions without you in the room.
- **Templates.** Provide design doc templates that guide teams through the right questions.
- **Office hours.** Set a weekly slot where any team can bring a design question. This is more efficient than ad-hoc meetings.
- **Trusted reviewers.** Identify senior engineers on each team who can review designs on your behalf. Invest in calibrating their judgment with yours.
