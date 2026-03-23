# 01 - Introduction to Software Engineering

## Concepts

### What is Software Engineering?

Software engineering is the disciplined application of engineering principles to the design, development, testing, deployment, and maintenance of software systems. It goes beyond writing code.

**Programming** is writing instructions for a computer to execute. You solve a problem with code.

**Computer Science** is the study of computation, algorithms, data structures, and the theoretical foundations of what computers can do.

**Software Engineering** is applying systematic, repeatable processes to build software that is reliable, scalable, maintainable, and delivered on time and within budget. It's about building software *as a team, over time, at scale*.

> "Programming is about making things work. Software engineering is about making things work *reliably, repeatedly, and collaboratively* — while the requirements keep changing."

The key distinction: a single developer can program. Software engineering is what happens when a team of people must build, evolve, and maintain a system over years.

### Software Development Life Cycle (SDLC)

The SDLC is a framework that describes the stages involved in building software. Every software project goes through these phases — the difference is *how* they're organized.

#### Waterfall

A sequential, phase-by-phase approach: Requirements → Design → Implementation → Testing → Deployment → Maintenance. Each phase must complete before the next begins.

**How it works in practice:**
- A business analyst writes a 200-page requirements document
- Architects create detailed design specs
- Developers implement exactly what's specified
- QA tests against the original requirements
- The product ships months (or years) later

**Where it's still used:** Government contracts, regulated industries (medical devices, aviation), projects with fixed requirements that truly won't change.

**Why it often fails:** Requirements change. A product built over 18 months may be outdated by the time it ships. Feedback comes too late to incorporate.

#### Iterative & Incremental

Build the system in small increments, getting feedback after each iteration. Each iteration produces a working (but incomplete) version of the system.

**How it works in practice:**
- Build a minimal version of the most important feature
- Demo it, get feedback
- Adjust and build the next increment
- Repeat until the product is complete

#### Spiral

Combines iterative development with risk analysis. Each loop through the spiral involves: planning, risk analysis, engineering, and evaluation. Used for large, complex, high-risk projects.

#### Agile

A family of methodologies (Scrum, Kanban, XP) based on the [Agile Manifesto](https://agilemanifesto.org/) (2001):

- **Individuals and interactions** over processes and tools
- **Working software** over comprehensive documentation
- **Customer collaboration** over contract negotiation
- **Responding to change** over following a plan

**Scrum** organizes work into 1-4 week sprints with defined roles (Product Owner, Scrum Master, Development Team) and ceremonies (sprint planning, daily standup, sprint review, retrospective).

**Kanban** visualizes work on a board with columns (To Do, In Progress, Done) and limits work-in-progress (WIP) to prevent bottlenecks.

**Extreme Programming (XP)** emphasizes pair programming, continuous integration, TDD, and frequent releases.

**How companies actually do it:** Most teams use a hybrid. They might run Scrum sprints but use a Kanban board. They write user stories but skip formal ceremonies when they slow things down. Pure agile is rare — pragmatic agile is the norm.

### Roles in a Software Team

| Role | Responsibility | Why it exists |
|------|---------------|---------------|
| **Software Engineer / Developer** | Designs, writes, tests, and maintains code | Builds the product |
| **QA Engineer** | Designs test strategies, writes automated tests, finds defects | Ensures quality before users see bugs |
| **Product Manager (PM)** | Defines what to build and why, prioritizes features | Bridges business needs and engineering |
| **Engineering Manager (EM)** | Manages people, removes blockers, grows the team | Ensures the team is healthy and productive |
| **Tech Lead** | Makes technical decisions, sets coding standards, mentors | Ensures technical quality and consistency |
| **DevOps / Platform Engineer** | Builds CI/CD pipelines, manages infrastructure | Enables fast, reliable deployments |
| **Site Reliability Engineer (SRE)** | Ensures system uptime, manages incidents, sets SLOs | Keeps the product running in production |
| **Designer (UX/UI)** | Designs user experiences and interfaces | Ensures the product is usable |
| **Data Engineer** | Builds data pipelines, manages data infrastructure | Enables data-driven decisions |
| **Security Engineer** | Identifies vulnerabilities, designs secure systems | Protects the product and its users |

In startups, one person might wear multiple hats. In large companies, these roles are distinct and specialized.

### How Software Creates and Captures Value

Software creates value in four fundamental ways:

1. **Automation** — Replacing manual work with software. Example: An invoicing system that generates and sends 10,000 invoices per month, replacing 3 full-time accountants.

2. **Enabling new capabilities** — Making things possible that weren't before. Example: Real-time GPS navigation, streaming video, instant messaging across the globe.

3. **Improving decision-making** — Turning data into insights. Example: A recommendation engine that increases e-commerce conversion by 15%.

4. **Reducing friction** — Making existing processes faster, cheaper, or more pleasant. Example: Mobile check deposit replacing a trip to the bank.

Software captures value through:
- **Direct revenue** — Subscriptions (SaaS), licenses, transactions
- **Cost reduction** — Fewer employees, less manual work, fewer errors
- **Competitive advantage** — Features competitors don't have
- **Network effects** — Each new user makes the product more valuable (social networks, marketplaces)

## Business Value

Understanding software engineering as a discipline (not just coding) directly impacts business outcomes:

- **Reduced project failure rates**: The Standish Group's CHAOS report consistently shows that projects following SE practices succeed at 2-3x the rate of ad-hoc development.
- **Predictable delivery**: Structured SDLC models enable realistic planning and deadline commitments — critical for go-to-market timing.
- **Lower total cost of ownership**: Systems built with engineering discipline cost less to maintain over their lifetime. Maintenance accounts for 60-80% of total software cost.
- **Scalable teams**: SE practices (code review, testing, documentation) allow teams to grow from 5 to 50 engineers without chaos.
- **Reduced risk**: Systematic approaches catch issues early when they're cheap to fix, rather than in production when they're expensive.

## Real-World Examples

### Spotify's Squad Model
Spotify organized their engineering teams into autonomous "squads" (small cross-functional teams), grouped into "tribes" (collections of related squads). Each squad owns a feature area end-to-end. This structure allowed Spotify to scale from a small startup to 1,000+ engineers while maintaining fast delivery. The model addresses a core SE challenge: how do you coordinate hundreds of engineers building the same product?

### Healthcare.gov Launch (2013)
The initial launch of Healthcare.gov is a textbook example of what happens when SE principles are ignored at scale. The project involved 55 contractors with no unified technical leadership, no integrated testing, and a waterfall approach with a hard political deadline. The result: the site crashed on day one, handling only 1,100 simultaneous users instead of the expected 50,000. The recovery effort applied proper SE practices — a small empowered team, incremental delivery, continuous testing — and stabilized the site within two months.

### Toyota Production System → Lean Software Development
Toyota's manufacturing principles (eliminate waste, build quality in, deliver fast, respect people) were adapted to software by Mary and Tom Poppendieck. Companies like Amazon adopted these ideas: small teams, two-pizza rule, "you build it, you run it" ownership. Amazon's move from monolith to microservices was driven by these SE organizational principles, enabling them to deploy thousands of times per day.

### How Google Does SDLC
Google uses a blend of agile practices adapted to their scale. Teams use OKRs (Objectives and Key Results) for planning, design docs for technical proposals, and extensive code review (every change must be reviewed by at least one other engineer). Their monorepo (a single repository for billions of lines of code) is supported by custom tooling (Blaze/Bazel, Critique, Piper). This demonstrates that SDLC at scale requires both process discipline and tooling investment.

## Common Mistakes & Pitfalls

- **"We don't need process, we're agile"** — Agile is not the absence of process. It's a different kind of process. Teams that interpret agile as "no planning, no documentation" end up with chaos, not agility.

- **Cargo-culting methodologies** — Adopting Scrum ceremonies (standups, sprints, retros) without understanding *why* they exist. A daily standup where everyone gives status reports to a manager is not a standup — it's a status meeting.

- **Over-engineering the SDLC** — Implementing heavyweight processes for a 3-person startup. A team of 3 doesn't need Jira workflows with 12 ticket states. They need a shared TODO list and regular conversations.

- **Ignoring the human side** — Focusing only on code and tools while ignoring team dynamics, communication, and psychological safety. The best processes fail if people don't trust each other.

- **One-size-fits-all thinking** — Assuming the process that worked at your last company will work at your current one. Context matters: team size, product maturity, regulatory requirements, and company culture all influence which practices fit.

## Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **Waterfall** | Clear milestones, easy to manage contracts, good for fixed requirements | Late feedback, expensive changes, assumes perfect foresight |
| **Agile** | Fast feedback, adapts to change, high team morale | Harder to predict long-term timelines, requires customer involvement, can drift without vision |
| **Heavy process** | Consistency, compliance, audit trails | Slow, demotivating, fights against creativity |
| **Minimal process** | Fast, flexible, low overhead | Breaks down at scale, knowledge silos, hard to onboard new people |

The right balance depends on your context. A medical device company building pacemaker firmware needs more process than a startup testing a new social feature.

## When to Use / When Not to Use

**When SE practices are essential:**
- Teams larger than 3-5 people
- Systems that will be maintained for more than 6 months
- Products where reliability matters (finance, healthcare, infrastructure)
- When onboarding new team members regularly
- When multiple teams must coordinate

**When you can afford to be lightweight:**
- Solo projects or hackathons
- Prototypes and proof-of-concepts (but be honest about when a prototype becomes a product)
- Short-lived scripts or tools with a single user
- Early-stage exploration where the goal is learning, not building

## Key Takeaways

1. Software engineering is programming + process + people + time. The "engineering" part is what makes software sustainable.
2. SDLC models are frameworks, not religions. Adapt them to your context.
3. Every role on a software team exists to solve a specific coordination problem. Understand why each role matters before deciding you don't need it.
4. Software creates value through automation, new capabilities, better decisions, and reduced friction. Every engineering decision should trace back to one of these.
5. The best process is the lightest process that prevents the problems you actually have. Start minimal, add process when pain appears.

## Further Reading

- **Books:**
  - *Software Engineering at Google* — Titus Winters, Tom Manshreck, Hyrum Wright (2020) — How Google approaches SE at massive scale
  - *The Mythical Man-Month* — Fred Brooks (1975) — Timeless lessons on why adding people to a late project makes it later
  - *Lean Software Development* — Mary & Tom Poppendieck (2003) — Applying lean manufacturing principles to software

- **Papers & Articles:**
  - [No Silver Bullet — Essence and Accident in Software Engineering](http://worrydream.com/refs/Brooks-NoSilverBullet.pdf) — Fred Brooks (1986)
  - [The Agile Manifesto](https://agilemanifesto.org/) — The original document that started the agile movement
  - [Spotify Engineering Culture](https://engineering.atspotify.com/) — Blog posts and videos on how Spotify organizes engineering

- **Talks:**
  - *Software Engineering at Google* — Hyrum Wright (various conference talks)
  - *The Art of Organizational Manipulation* — James Mickens (Monitorama 2018) — A humorous but insightful talk on how organizations actually work
