# Technical Strategy

Technical strategy is the bridge between business goals and engineering execution. It answers
the question: given where the business wants to go, how should we build our systems to get
there efficiently? A Principal Engineer owns this strategy for the engineering organization.

Getting this right is one of the highest-leverage activities in all of engineering. A good
technical strategy aligns the work of dozens of teams without requiring constant coordination.
A bad one — or the absence of one — leads to duplicated effort, incompatible systems, and an
engineering organization that feels busy but cannot explain why the product is not moving
faster.

This subtopic covers how to build, communicate, and maintain a technical strategy that
actually drives decisions. It is not about frameworks or templates — it is about the thinking
process that produces a strategy worth following.

---

## What Technical Strategy Is Not

Before defining what a technical strategy is, it helps to clear away common misconceptions
that lead to ineffective documents.

- **Not a technology wishlist.** "We should adopt Kubernetes" is not a strategy. "We need to
  reduce deployment friction to ship features 3x faster, and container orchestration is one
  component of that" is a strategy. The distinction matters: a wishlist is a list of tools;
  a strategy connects tools to outcomes.
- **Not a roadmap.** A roadmap lists what you will build and when. A strategy explains *why*
  you are building it and *how* the pieces fit together. Roadmaps change quarterly. Strategy
  evolves more slowly because it operates at a higher level of abstraction.
- **Not permanent.** Strategy evolves as the business, market, and technology landscape
  change. Review it quarterly. But if your strategy changes dramatically every quarter, what
  you have is not a strategy — it is a series of reactions.
- **Not a consensus document.** A good strategy makes choices. Choices create losers — teams
  whose preferred approach was not selected, technologies that are being deprecated. If
  everyone is happy with your strategy, it probably does not say anything meaningful.

## Building a Technical Strategy

### Step 1: Understand the Business Context

You cannot set technical direction without understanding where the business is going. Too
many technical strategies fail because they are built in isolation from business reality.

Key questions to answer:

- What are the company's growth targets for the next 1-3 years?
- What new products or markets are planned?
- What are the biggest business risks?
- Where is the company spending the most money on engineering, and is it getting good ROI?
- What is the competitive landscape and how does technology factor into differentiation?

Get this from the CTO, VP of Engineering, and product leadership. If they do not have clear
answers, that is a problem you should flag — and help resolve, because you cannot build
strategy on a foundation of ambiguity.

A concrete example: a fintech company planned to expand from the US to 12 European markets
within two years. The Principal Engineer discovered that the payment processing system
hard-coded USD assumptions in 340 places, the compliance engine had no concept of GDPR, and
the deployment pipeline assumed a single AWS region. Without understanding this business
context, the technical strategy would have focused on performance optimization — work that
was important but not existential. With the context, the strategy correctly prioritized
internationalization infrastructure.

### Step 2: Assess the Current State

Audit the technical landscape honestly. This is where many strategies fail — either because
the assessment is too rosy (ignoring real problems) or too negative (creating a crisis
narrative that undermines confidence).

- **Architecture.** What are the major systems? How do they interconnect? Where are the pain
  points? Draw the actual architecture, not the intended one.
- **Technical debt.** What is slowing teams down? What is fragile? What is one incident away
  from a major outage? Quantify where possible — "the deployment pipeline fails 15% of the
  time" is more actionable than "deployments are unreliable."
- **Team capabilities.** What technologies does the organization know well? Where are the
  skill gaps? A strategy that requires deep Rust expertise in an organization of Python
  developers is not realistic without a multi-quarter hiring and training plan.
- **Tooling & infrastructure.** Is the build/deploy pipeline fast and reliable? Can teams
  move independently? How long does it take a new engineer to ship their first change?
- **Data landscape.** Where does data live? How does it flow? Are there data quality issues
  that affect product decisions?

A useful technique is to interview 10-15 team leads with the same set of questions and look
for patterns. If five teams independently name the same pain point, that is a strong signal.
If only one team mentions something, it might be a local problem rather than a strategic one.

### Step 3: Identify Strategic Themes

Group your findings into 3-5 themes that connect business needs to technical investments.
More than five themes means you have not prioritized. Fewer than three means you are probably
not being comprehensive enough.

**Example themes from a real e-commerce company:**

- **Scale for 10x growth.** Rearchitect the order pipeline to handle projected transaction
  volume without increasing infrastructure cost linearly. Current cost-per-transaction is
  $0.12; target is $0.03.
- **Accelerate delivery.** Reduce the average time from PR merge to production from 4 hours
  to 15 minutes. Current bottleneck is a sequential integration test suite that takes 90
  minutes and flakes 20% of the time.
- **Reduce operational burden.** Consolidate from 5 deployment platforms to 1, freeing 20%
  of SRE time for proactive reliability work. Currently, 3 of 8 SREs spend their entire
  week managing deployment infrastructure.

Each theme should have:

- A clear business justification (why it matters to revenue, cost, or risk)
- A measurable target (how you know it worked)
- A rough scope and timeline (quarters, not days)
- Dependencies and sequencing (which themes must come first)

### Step 4: Define Principles

Principles are the decision-making guidelines for cases the strategy does not explicitly
cover. They should be specific enough to actually resolve disagreements.

Bad principle: "We value simplicity." (Everyone agrees, so it resolves nothing.)

Good principle: "When choosing between a custom solution and a managed service, default to
the managed service unless the custom solution provides a measurable competitive advantage."
This principle actually changes behavior — it tells the team that wanted to build a custom
message queue to use Amazon SQS instead.

More examples of effective principles:

```text
"All data stores must have a designated owning team.
 No shared databases between services."

"New services must be deployable independently within their
 first sprint. If a service requires coordinated deployment
 with another service, the architecture is wrong."

"We prefer boring technology for infrastructure and reserve
 novel technology for product differentiation."
```

Aim for 5-8 principles. Each should be opinionated enough that a reasonable engineer could
disagree with it. If nobody would disagree, the principle is not saying anything useful.

### Step 5: Write It Down

A technical strategy document should be readable by both engineers and executives. Structure
it as:

1. **Context** — Business goals and constraints (1-2 pages)
2. **Current state** — Honest assessment with data (2-3 pages)
3. **Strategic themes** — The 3-5 big bets with justification (3-5 pages)
4. **Principles** — Decision-making guidelines (1 page)
5. **What we are NOT doing** — Equally important. Name the investments you considered and
   rejected, with reasoning (1-2 pages)

The "what we are NOT doing" section is often the most valuable part. It prevents teams from
spending cycles on work that has already been considered and deprioritized.

Example:

```text
NOT DOING: Rewrite the monolith in Go.
REASON:    Rewrite risk is too high given current team composition.
           Performance gains do not justify the 12-month investment
           when targeted optimizations can achieve 80% of the benefit.
           Revisit if Go expertise increases through hiring.
```

### Step 6: Socialize & Iterate

Share the draft widely. Present it to engineering leadership, then to the broader
organization. Incorporate feedback. The strategy is only useful if people understand it,
believe in it, and use it to guide their decisions.

Effective socialization follows a pattern:

1. **Inner circle review** — Share with 3-5 trusted Staff Engineers and your engineering
   leadership peers. Incorporate their feedback before going broader.
2. **Leadership alignment** — Present to the CTO and VP of Engineering. Ensure the strategy
   is aligned with their expectations and get explicit buy-in.
3. **Broad engineering presentation** — Present to all engineers in a town hall format.
   Record it. Make the document available.
4. **Team-level workshops** — Help each team understand how the strategy applies to their
   domain. This is where abstract themes become concrete plans.

The socialization step is where most strategies fail. A brilliant document that lives in a
wiki page nobody reads is indistinguishable from no strategy at all.

## Maintaining the Strategy

A strategy written once and forgotten is worse than no strategy — it creates false confidence.
Establish a quarterly review cadence:

- Are the themes still aligned with business priorities?
- Are teams making decisions consistent with the strategy?
- Have new constraints or opportunities emerged?
- What has the organization learned from executing against the strategy?
- Which bets are paying off and which need adjustment?

Update the document and communicate changes. Stability is good — a strategy that changes
every month is not a strategy. But stubbornness is bad — ignoring new information because it
contradicts your plan is how organizations fail.

### Measuring Strategy Effectiveness

Track whether the strategy is actually being used:

- **Decision alignment.** In design reviews, do teams reference the strategy when justifying
  choices? If not, the strategy is either unknown or irrelevant.
- **Theme progress.** Are the measurable targets improving? If the strategy says "reduce
  deployment time to 15 minutes" and it is still 4 hours after two quarters, something is
  broken — either the strategy is wrong, the execution is blocked, or the investment is
  insufficient.
- **Principle violations.** When teams deviate from principles, is it intentional and
  reasoned, or accidental and uninformed? A few intentional exceptions are healthy.
  Widespread ignorance is a socialization failure.
- **Team feedback.** Do engineers find the strategy useful? Do they reference it in design
  docs and proposals? Ask directly in team retros and engineering surveys.

## Real-World Example: Strategy at a Growing Startup

A B2B SaaS company with 80 engineers was struggling with reliability. Outages were weekly,
on-call was burning people out, and customer churn was increasing. The previous "strategy"
was a list of 23 technology initiatives with no prioritization.

The incoming Principal Engineer spent four weeks on assessment:

- Interviewed 15 team leads about their biggest pain points
- Analyzed 6 months of incident reports and identified that 70% originated from 3 services
- Measured deployment frequency (2 per week, down from 10 the previous year) and change
  failure rate (35%)
- Mapped the architecture and found 12 circular service dependencies

The resulting strategy had three themes:

1. **Stabilize the foundation.** Fix the 3 services causing 70% of incidents. Eliminate
   circular dependencies. Target: reduce incident rate by 60% in two quarters.
2. **Restore deployment confidence.** Implement canary deployments, automated rollback, and
   feature flags. Target: return to 10+ deployments per week with a change failure rate
   below 5%.
3. **Enable team autonomy.** Break the shared database into domain-owned data stores so teams
   can deploy independently. Target: zero cross-team deployment coordination required.

The "NOT doing" section explicitly listed: no new microservices until stability improves, no
migration to a new cloud provider, no adoption of Kubernetes (the current infrastructure was
not the bottleneck).

Within two quarters, incident rate dropped 55% and deployment frequency tripled. The strategy
worked because it was grounded in data, focused on the real problems, and explicitly rejected
appealing distractions.

## Common Pitfalls

- **Strategy by committee.** Soliciting feedback is important. Designing the strategy by
  consensus produces a document that offends no one and inspires no one. The Principal
  Engineer must own the strategy and make hard calls.
- **All vision, no execution path.** A strategy that describes a beautiful future state
  without explaining how to get there is a fantasy. Include concrete first steps for each
  theme.
- **Ignoring organizational reality.** A strategy that requires capabilities the organization
  does not have — and has no plan to acquire — will fail. Account for team skills, hiring
  timelines, and organizational appetite for change.
- **Overindexing on technology trends.** Adopting a technology because it is popular rather
  than because it solves a real problem. Every technology in the strategy should trace back
  to a business need.
- **Failing to kill the old strategy.** When you publish a new strategy, explicitly retire
  the old one. Otherwise, people cite whichever version supports their preferred approach.
- **No feedback loop.** Publishing a strategy and never measuring whether it is working. If
  you do not track progress against your themes, you cannot course-correct.
- **Perfectionism.** Waiting six months to publish a polished strategy means six months of
  teams making unguided decisions. A good-enough strategy published now beats a perfect
  strategy published next quarter.

## Key Takeaways

- Technical strategy connects business goals to engineering execution — it is not a technology
  wishlist or a roadmap
- Start with business context, not technology preferences; the best strategy solves the
  company's actual problems
- Limit strategic themes to 3-5 with measurable targets and clear business justification
- The "what we are NOT doing" section is often the most valuable part of the document
- Principles should be opinionated enough to actually resolve disagreements between
  reasonable engineers
- Socialization is as important as writing — a strategy that no one reads or understands has
  zero value
- Review quarterly, update when warranted, but resist the urge to change direction based on
  every new input
- Measure effectiveness by tracking decision alignment, theme progress, and principle adoption
  across teams
