# Microservices vs Monolith

## Two Architectural Models

A monolith is a single deployable unit containing all of an application's functionality. A microservices architecture splits that functionality into many small, independently deployable services, each owning a specific business domain.

Neither is inherently better. The right choice depends on your team size, organizational structure, operational maturity, and scaling requirements. Most failed architecture decisions come from choosing microservices too early or sticking with a monolith too long.

## The Monolith

All code lives in one repository, compiles into one artifact, and deploys as one unit.

```
┌─────────────────────────────────────────┐
│              Monolith                    │
│                                         │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │  Users    │ │  Orders  │ │ Payment │ │
│  │  Module   │ │  Module  │ │ Module  │ │
│  └────┬─────┘ └────┬─────┘ └────┬────┘ │
│       │             │            │       │
│       └─────────────┼────────────┘       │
│                     │                    │
│              ┌──────┴──────┐             │
│              │  Shared DB  │             │
│              └─────────────┘             │
└─────────────────────────────────────────┘
```

**Advantages:**
- Simple to develop, test, and deploy — one build, one artifact, one deployment
- In-process function calls between modules (nanoseconds, not milliseconds)
- ACID transactions across all data — no distributed transaction headaches
- Easy to debug — a single stack trace tells the whole story
- One set of infrastructure to manage (one CI pipeline, one monitoring setup, one logging destination)

**Disadvantages:**
- Deployment coupling — changing one line in the payment module requires redeploying the entire application
- Scaling is all-or-nothing — if the search module needs 10x more CPU, you scale the entire monolith 10x
- Team coupling — 50 engineers committing to the same codebase creates merge conflicts, broken builds, and coordination overhead
- Technology lock-in — every module must use the same language, framework, and database
- Reliability risk — a memory leak in one module can crash the entire application

## Microservices

Each service is independently developed, deployed, and scaled. Services communicate over the network via APIs or events.

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Users   │     │  Orders  │     │ Payment  │
│ Service  │────→│ Service  │────→│ Service  │
│          │     │          │     │          │
│ [Own DB] │     │ [Own DB] │     │ [Own DB] │
└──────────┘     └──────────┘     └──────────┘
      ↑                ↑                ↑
      └────────────────┼────────────────┘
                       │
                 [API Gateway]
                       │
                    [Client]
```

**Advantages:**
- Independent deployment — the payment team deploys without coordinating with the orders team
- Independent scaling — scale the search service to 20 instances while the user profile service runs on 2
- Technology flexibility — the recommendation service can use Python with ML libraries while the real-time service uses Rust
- Fault isolation — the payment service crashing does not take down the product catalog
- Team autonomy — each team owns their service end-to-end (code, data, deployment, monitoring)

**Disadvantages:**
- Distributed system complexity — network failures, latency, partial failures, eventual consistency
- Operational overhead — each service needs its own CI/CD pipeline, monitoring, alerting, logging
- Data consistency is hard — no cross-service ACID transactions. You use sagas, eventual consistency, or compensating transactions
- Debugging is hard — a single user request may span 10 services. You need distributed tracing (Jaeger, Zipkin)
- Latency overhead — in-process calls (nanoseconds) become network calls (milliseconds)
- Integration testing is painful — testing the interaction between 20 services requires a complex test environment

## Detailed Comparison

| Factor | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | All-or-nothing | Independent per service |
| **Scaling** | Entire application | Individual services |
| **Development speed (small team)** | Faster | Slower (overhead) |
| **Development speed (large org)** | Slower (coordination) | Faster (autonomy) |
| **Data consistency** | ACID transactions | Eventual consistency, sagas |
| **Debugging** | Stack trace | Distributed tracing |
| **Latency** | In-process (ns) | Network calls (ms) |
| **Operational cost** | Low | High |
| **Technology diversity** | Single stack | Polyglot |
| **Team independence** | Low | High |
| **Failure blast radius** | Entire application | Single service (if designed well) |

## Conway's Law

> "Any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure." — Melvin Conway, 1967

This is not just an observation — it is a force of nature in software. If you have three teams, you will get three services (or three major modules). If all engineers sit together and communicate freely, you will naturally build a monolith. If teams are distributed and autonomous, they will naturally build separate services.

**The practical implication:** Do not choose microservices and then try to run them with a monolith team structure. And do not try to force a monolith on an organization of 20 autonomous teams. Align your architecture with your organization.

**The inverse Conway maneuver:** Some organizations deliberately restructure teams to produce the architecture they want. Want microservices? Create small, autonomous teams each owning a bounded context. The architecture follows.

## When to Migrate from Monolith to Microservices

Migration is justified when:

1. **Teams are stepping on each other.** Merge conflicts are constant, deploys require coordination across 5+ teams, and a bug in one team's code blocks everyone's deployment.
2. **Components have different scaling needs.** The search feature needs 50 servers while the admin panel needs 1, but you are scaling everything together.
3. **Deployment frequency is suffering.** You want to deploy daily but the monolith requires a 2-hour regression test, so you deploy weekly.
4. **Fault isolation is critical.** A bug in the recommendation engine should not crash the checkout flow.
5. **You have the operational maturity.** You have CI/CD, automated testing, monitoring, distributed tracing, and engineers who understand distributed systems.

**Do NOT migrate when:**
- You have fewer than 20-30 engineers. The operational overhead of microservices will slow you down.
- You do not have automated deployment pipelines. Manual deployments times N services equals pain.
- You are trying to solve a code quality problem. Microservices do not fix bad code; they just distribute it.
- You are in the early stages of product development. You do not yet know where the service boundaries should be.

## The Migration Path

### The Strangler Fig Pattern

The safest migration strategy. Named after strangler fig trees that grow around a host tree and eventually replace it.

```
Phase 1: Monolith handles everything
┌────────────────────────┐
│       Monolith         │
│  [Users] [Orders] [Pay]│
└────────────────────────┘

Phase 2: New service handles one domain, monolith proxies to it
┌────────────────────────┐     ┌──────────┐
│       Monolith         │────→│  Orders  │
│  [Users] [-----] [Pay] │     │ Service  │
└────────────────────────┘     └──────────┘

Phase 3: More domains extracted
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Users   │  │  Orders  │  │ Payment  │
│ Service  │  │ Service  │  │ Service  │
└──────────┘  └──────────┘  └──────────┘
```

Steps:
1. Identify a bounded context with clear boundaries (e.g., the orders domain)
2. Build the new service alongside the monolith
3. Route traffic for that domain to the new service (via API gateway or proxy)
4. Once stable, remove the dead code from the monolith
5. Repeat for the next domain

### Data Extraction

The hardest part of migration is extracting data. In a monolith, modules often share database tables through JOINs. Extracting a service means:

1. Identify which tables belong to the new service
2. Create a new database for the service
3. Migrate data to the new database
4. Replace direct JOINs with API calls between services
5. Set up change data capture or events for data that was previously joined

This is where most migrations stall. Shared database tables are the strongest coupling in a monolith.

## API Gateways

An API gateway is a single entry point for all client requests to a microservices backend. Without one, clients must know the address of every service and handle cross-cutting concerns themselves.

```
Mobile App ──→ ┌──────────────┐ ──→ User Service
Web App   ──→ │  API Gateway  │ ──→ Order Service
Partner   ──→ └──────────────┘ ──→ Payment Service
```

**Responsibilities:**
- **Routing:** Direct `/api/users/*` to the user service, `/api/orders/*` to the order service
- **Authentication:** Verify JWTs or API keys once at the gateway instead of in every service
- **Rate limiting:** Protect backend services from abuse (token bucket, sliding window)
- **Request transformation:** Translate between external API format and internal service formats
- **Response aggregation:** Combine responses from multiple services into one client response
- **SSL termination:** Handle HTTPS at the edge so internal traffic can be plain HTTP
- **Observability:** Centralized request logging, metrics, and tracing header injection

**Popular API gateways:**
- **Kong:** Open source, plugin-based, built on NGINX. Widely adopted.
- **AWS API Gateway:** Fully managed, integrates with Lambda and other AWS services
- **Envoy:** High-performance proxy often used as both API gateway and service mesh data plane
- **NGINX:** Can serve as a simple API gateway with reverse proxy configuration

### BFF Pattern (Backend for Frontend)

Instead of one generic API gateway, create a dedicated backend for each client type (mobile BFF, web BFF, partner BFF). Each BFF tailors responses for its client — the mobile BFF returns smaller payloads, the web BFF returns richer data. This avoids the "one-size-fits-none" problem of a generic API.

## Service Mesh

A service mesh is a dedicated infrastructure layer for service-to-service communication. It handles networking concerns (load balancing, retries, circuit breaking, mutual TLS, observability) without changing application code.

```
┌────────────────────┐           ┌────────────────────┐
│   Service A        │           │   Service B        │
│  ┌──────────────┐  │           │  ┌──────────────┐  │
│  │  App Code    │  │           │  │  App Code    │  │
│  └──────┬───────┘  │           │  └──────┬───────┘  │
│         │          │           │         │          │
│  ┌──────┴───────┐  │  network  │  ┌──────┴───────┐  │
│  │ Sidecar Proxy│◄─┼──────────►┼─►│ Sidecar Proxy│  │
│  └──────────────┘  │           │  └──────────────┘  │
└────────────────────┘           └────────────────────┘
            ▲                               ▲
            └───────── Control Plane ───────┘
```

### How It Works

Every service instance gets a sidecar proxy (a small network proxy running alongside the application). All inbound and outbound traffic passes through the sidecar. The sidecar handles:

- **Mutual TLS (mTLS):** Encrypts all service-to-service traffic and verifies identities. Zero-trust networking without application changes.
- **Load balancing:** Intelligent, client-side load balancing with health checking
- **Retries and timeouts:** Automatic retries with exponential backoff for transient failures
- **Circuit breaking:** If Service B is failing, the sidecar stops sending traffic to it (preventing cascade failures)
- **Observability:** Every request is traced, metriced, and logged by the sidecar

A control plane manages all sidecars: distributing configuration, collecting telemetry, enforcing policies.

### Istio vs Linkerd

| Feature | Istio | Linkerd |
|---------|-------|---------|
| **Proxy** | Envoy (C++, feature-rich) | linkerd2-proxy (Rust, lightweight) |
| **Complexity** | High (many configuration options) | Low (opinionated, fewer knobs) |
| **Resource overhead** | Higher (Envoy is heavier) | Lower (Rust proxy is minimal) |
| **Features** | Comprehensive (traffic management, security, observability) | Focused (reliability, observability, mTLS) |
| **Learning curve** | Steep | Gentle |
| **Best for** | Complex environments needing fine-grained control | Teams wanting a service mesh without the complexity |

**When to use a service mesh:**
- You have 20+ services and managing retries, timeouts, and mTLS in application code is unsustainable
- You need zero-trust networking (mTLS everywhere) without changing application code
- You want consistent observability across services written in different languages

**When NOT to use a service mesh:**
- You have fewer than 10 services — the overhead is not justified
- Your team is not comfortable with Kubernetes (service meshes assume Kubernetes)
- You can handle cross-cutting concerns with a shared library

## Real-World Migration Stories

### Netflix: The 7-Year Migration

Netflix began migrating from a monolithic Java application to microservices in 2008. The migration took approximately 7 years to complete (finishing around 2015).

**Why they migrated:** A database corruption incident in 2008 took down their entire service for 3 days. The monolith had a single Oracle database as a single point of failure. They could not scale individual components, and a failure anywhere brought down everything.

**How they did it:**
- Used the strangler fig pattern — new features were built as services, old features were gradually extracted
- Built an entire platform of tools: Zuul (API gateway), Eureka (service discovery), Hystrix (circuit breaker), Ribbon (client-side load balancing)
- Moved from Oracle to a mix of Cassandra, DynamoDB, and other purpose-built databases
- Developed the "Simian Army" — Chaos Monkey and related tools that randomly kill services in production to ensure resilience

**The cost:** Years of engineering effort, building an entire microservices platform from scratch, and operational complexity that requires a dedicated platform engineering team.

**The payoff:** Netflix now deploys thousands of times per day, scales individual services independently, and has achieved the resilience to survive entire AWS region failures. They serve 250+ million subscribers with 99.99% uptime.

**Lesson:** Netflix had the engineering talent, the organizational scale (thousands of engineers), and the business need (global streaming at massive scale) to justify this migration. Most companies do not.

### Amazon's API Mandate

In 2002, Jeff Bezos issued a now-famous mandate (paraphrased):
1. All teams will expose their data and functionality through service interfaces
2. Teams must communicate with each other through these interfaces
3. All service interfaces must be designed to be externalizable (usable by the outside world)
4. Anyone who does not do this will be fired

This mandate forced Amazon from a monolithic architecture to service-oriented architecture years before "microservices" was a term. The result was AWS itself — the internal services Amazon built (compute, storage, databases) were so well-designed for external use that they became products.

**Key insight:** Amazon's migration was driven by organizational structure as much as technical need. The mandate aligned the architecture with autonomous "two-pizza teams" (teams small enough to be fed by two pizzas). Conway's Law in action.

### Shopify: The Modular Monolith

Not every migration story ends in microservices. Shopify took a different path: instead of splitting into microservices, they restructured their monolithic Ruby on Rails application into a modular monolith.

**What they did:**
- Defined clear module boundaries within the monolith (using a system they call "components")
- Enforced that modules communicate through defined interfaces, not by reaching into each other's internals
- Each module owns its database tables — no cross-module JOINs
- Modules can be extracted into services later if needed, but many never will be

**Why this worked for Shopify:**
- They get most of the organizational benefits of microservices (team ownership, clear boundaries) without the operational cost
- In-process communication means no network latency between modules
- ACID transactions across modules remain possible
- Deployment is still one artifact, but modules are independently testable

**Lesson:** The modular monolith is an underappreciated middle ground. It gives you clean boundaries and team ownership without the distributed system tax.

## Common Pitfalls

- **Distributed monolith.** You split into microservices but they all share a database, deploy together, and cannot function independently. You have the worst of both worlds: the complexity of microservices with the coupling of a monolith.
- **Too many services too early.** Starting a greenfield project with 15 microservices when you have 5 engineers. You will spend more time on infrastructure than product.
- **Ignoring data ownership.** If two services share database tables, they are not independent services. Data ownership is the foundation of microservice independence.
- **Synchronous chains.** Service A calls B, which calls C, which calls D. Latency adds up, and any failure in the chain fails the entire request. Use asynchronous events where possible.
- **No API versioning.** Changing a service's API without versioning breaks all consumers. Version your APIs from day one.
- **Skipping the platform.** Microservices require platform capabilities: service discovery, centralized logging, distributed tracing, CI/CD per service. Without these, operations become unmanageable.

## Key Takeaways

1. Start with a monolith. Extract microservices when you have concrete evidence that you need them (team scaling, independent scaling, deployment frequency).
2. Conway's Law is real. Align your architecture with your organizational structure, or change your organization to match the architecture you want.
3. The modular monolith is a viable middle ground that is often overlooked.
4. The strangler fig pattern is the safest migration strategy. Never attempt a big-bang rewrite.
5. Data extraction is the hardest part of any migration. Shared databases are the strongest coupling.
6. API gateways and service meshes are essential infrastructure for microservices at scale, but add them when the pain justifies the complexity.
7. Operational maturity (CI/CD, monitoring, tracing) is a prerequisite for microservices, not something you figure out after.
