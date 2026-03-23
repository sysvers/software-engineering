# Software Engineering

A comprehensive guide to software engineering — from foundational practices to advanced topics. Covers what you'd learn in college and master's programs, but with real-world examples and use cases from actual companies.

Each topic covers:
- **Concepts** — the theory and principles behind the practice
- **Business Value** — how the practice drives revenue, efficiency, or growth
- **Real-World Examples** — how companies actually apply these practices
- **Common Mistakes & Pitfalls** — what teams get wrong in practice, anti-patterns to avoid
- **Trade-offs** — when does this practice help vs hurt
- **When to Use / When Not to Use** — applicability guidance
- **Key Takeaways** — quick summary for reference
- **Further Reading** — books, papers, talks, blog posts

When code examples are needed, they are written in **Rust** and **Svelte**.

## How to Use This Repo

Topics are numbered and ordered from foundational to advanced. Each topic lives in its own directory:

```
XX-topic-name/
└── README.md            # Concepts, business value, real-world examples,
                         # common mistakes, trade-offs, when to use,
                         # key takeaways, further reading
                         # (Rust code examples are inline)
```

Work through them in order, or jump to any topic that interests you.

---

## Topics

### Fundamentals

| # | Topic | Description |
|---|-------|-------------|
| 01 | [Introduction to Software Engineering](01-introduction-to-software-engineering/) | What is SE vs programming vs CS, SDLC (waterfall, iterative, spiral, agile), roles in a software team, how software creates and captures value |
| 02 | [Clean Code & Software Craftsmanship](02-clean-code-and-software-craftsmanship/) | Naming, readability, DRY/KISS/YAGNI, code smells & refactoring, linting (rustfmt, clippy), technical debt management |
| 03 | [Version Control & Collaboration](03-version-control-and-collaboration/) | Git internals (objects, refs, DAG), branching strategies (trunk-based, GitFlow, GitHub Flow), merge vs rebase, code review, monorepos vs polyrepos |
| 04 | [Documentation Engineering](04-documentation-engineering/) | Technical writing, API documentation, architecture docs (C4 model), runbooks & playbooks, README-driven development, docs-as-code, keeping docs in sync with code |
| 05 | [Requirements Engineering](05-requirements-engineering/) | Functional vs non-functional requirements, user stories, use cases, acceptance criteria, traceability, prototyping, SRS & PRD |
| 06 | [Unit & Integration Testing](06-unit-and-integration-testing/) | Test pyramid, TDD, unit testing strategies, integration testing, property-based testing (proptest), mocking & stubbing, test doubles, code coverage |
| 07 | [End-to-End & Performance Testing](07-e2e-and-performance-testing/) | E2E testing strategies, contract testing, mutation testing, fuzz testing, load testing, stress testing, soak testing, test environments management |
| 08 | [Debugging & Production Troubleshooting](08-debugging-and-production-troubleshooting/) | Systematic debugging methodology, debuggers (lldb/gdb), flamegraphs (cargo-flamegraph), memory profiling (valgrind, DHAT), production debugging (core dumps, tracing) |
| 09 | [Error Handling, Resilience & Fault Tolerance](09-error-handling-resilience-and-fault-tolerance/) | Rust error ecosystem (thiserror, anyhow), error hierarchies, retry strategies (exponential backoff, jitter), circuit breakers, bulkheads, timeouts, graceful degradation, chaos engineering |

### Design & Architecture

| # | Topic | Description |
|---|-------|-------------|
| 10 | [Design Patterns](10-design-patterns/) | Creational (Builder, Factory, Singleton), Structural (Adapter, Decorator, Facade, Proxy), Behavioral (Observer, Strategy, Command, State, Iterator), Rust-specific patterns (newtype, typestate), anti-patterns |
| 11 | [Low-Level System Design](11-low-level-system-design/) | Class/module design, interface design, concurrency design, state machines, data flow design, schema design for specific use cases, design walkthroughs |
| 12 | [API Design & Integration](12-api-design-and-integration/) | REST, GraphQL, gRPC & Protocol Buffers (tonic), API versioning, rate limiting & throttling, backpressure, OpenAPI/Swagger, webhooks, event-driven APIs |
| 13 | [Software Architecture](13-software-architecture/) | Layered, hexagonal, clean architecture, separation of concerns, dependency inversion, DDD (bounded contexts, aggregates, entities, value objects), modular monolith, ADRs |
| 14 | [Database Engineering](14-database-engineering/) | Schema design & migrations, connection pooling, ORMs vs raw SQL (sqlx, diesel), SQL deep dive (joins, window functions, CTEs), NoSQL trade-offs, data modeling for access patterns |
| 15 | [High-Level System Design](15-high-level-system-design/) | Vertical vs horizontal scaling, load balancing (L4 vs L7), caching strategies, message queues & event streaming (Kafka, RabbitMQ), microservices vs monolith, service mesh, API gateways, case studies |
| 16 | [Event-Driven Architecture & CQRS](16-event-driven-architecture-and-cqrs/) | Event sourcing (event store, projections, snapshots), CQRS, domain events vs integration events, event-driven microservices, eventual consistency patterns |

### Infrastructure & Operations

| # | Topic | Description |
|---|-------|-------------|
| 17 | [Build Systems, Tooling & Developer Experience](17-build-systems-tooling-and-dx/) | Cargo workspaces & features, build optimization, monorepo tooling (Bazel, Nx), dependency security (cargo-audit), developer productivity metrics (DORA, SPACE) |
| 18 | [Software Configuration Management](18-software-configuration-management/) | Feature flags & toggles, environment management, config as code, secrets management (Vault, SOPS), runtime configuration, A/B testing infrastructure, configuration drift detection |
| 19 | [CI/CD & Deployment](19-cicd-and-deployment/) | CI pipelines (build, test, lint, security scan), CD strategies (blue-green, canary, rolling), Kubernetes, Docker internals, GitOps, reproducible builds, artifact management |
| 20 | [Cloud Engineering & Infrastructure](20-cloud-engineering-and-infrastructure/) | Cloud-native design, serverless architecture, managed services vs self-hosted, multi-cloud strategies, networking (VPC, subnets, security groups), IaC (Terraform, Pulumi), cost optimization |
| 21 | [Observability & Monitoring](21-observability-and-monitoring/) | Logs, metrics, traces (three pillars), structured logging (tracing crate), Prometheus & Grafana, distributed tracing (OpenTelemetry), alerting strategies, dashboarding best practices |
| 22 | [Application Security](22-application-security/) | OWASP Top 10, authentication (sessions, JWT, OAuth 2.0, OIDC), authorization (RBAC, ABAC), secure coding, input validation, XSS/CSRF/SQL injection prevention, security testing (SAST, DAST) |
| 23 | [Infrastructure Security](23-infrastructure-security/) | Network security, secrets management, supply chain security, dependency auditing, container security, zero-trust architecture, security incident response, penetration testing |

### Advanced

| # | Topic | Description |
|---|-------|-------------|
| 24 | [Performance Engineering](24-performance-engineering/) | Benchmarking (criterion), CPU/memory/I/O profiling, optimization (zero-copy, SIMD, cache-friendly data structures), load testing (k6, locust), query optimization, Core Web Vitals |
| 25 | [Distributed Systems Engineering](25-distributed-systems-engineering/) | Service discovery, distributed tracing, data replication patterns, sharding strategies, distributed caching, idempotency, saga pattern, distributed transactions |
| 26 | [Site Reliability Engineering (SRE)](26-site-reliability-engineering/) | SLOs, SLIs, error budgets, capacity planning, incident response & management, on-call practices, post-mortems & blameless culture, toil reduction, reliability as a feature |
| 27 | [Data Engineering & Pipelines](27-data-engineering-and-pipelines/) | ETL vs ELT, batch vs stream processing, data warehousing (star schema, snowflake schema), data quality & governance, building data pipelines in Rust |
| 28 | [Legacy Systems & Migration](28-legacy-systems-and-migration/) | Working with legacy code, strangler fig pattern, incremental rewrites, migration strategies, backward compatibility, database migrations at scale, risk management during migration |
| 29 | [MLOps & AI Engineering](29-mlops-and-ai-engineering/) | ML pipeline (data prep, training, evaluation, deployment), feature stores, model versioning & serving, inference optimization, A/B testing, model drift monitoring, ethical AI |

### Professional & Governance

| # | Topic | Description |
|---|-------|-------------|
| 30 | [Software Project Management & Team Dynamics](30-software-project-management-and-team-dynamics/) | Agile (Scrum, Kanban, XP), estimation (story points, T-shirt sizing, Monte Carlo), tech debt prioritization, engineering metrics, cross-functional collaboration, technical writing (RFCs, design docs) |
| 31 | [Software Economics & Cost Engineering](31-software-economics-and-cost-engineering/) | Build vs buy decisions, total cost of ownership (TCO), FinOps & cloud cost management, ROI of engineering decisions, technical debt as financial debt, pricing engineering time |
| 32 | [Compliance & Regulatory Engineering](32-compliance-and-regulatory-engineering/) | GDPR, SOC2, HIPAA, PCI-DSS, audit trails, data residency & sovereignty, privacy by design, compliance automation, regulatory change management |
| 33 | [Open Source & Inner Source](33-open-source-and-inner-source/) | OSS licensing (MIT, Apache, GPL), governance models, community management, contributing to OSS, maintaining OSS projects, inner source practices within companies |
| 34 | [Career & Professional Growth](34-career-and-professional-growth/) | IC vs management track, code of ethics, open source contribution, mentoring & knowledge sharing, continuous learning strategies |

---

## Contributing

1. Fork the repo
2. Create a topic branch (`git checkout -b topic/XX-topic-name`)
3. Follow the directory structure above
4. Each topic must cover: concepts, business value, real-world examples, common mistakes & pitfalls, trade-offs, when to use / when not to use, key takeaways, and further reading
5. When code is needed, use Rust (and Svelte for frontend)
6. Submit a PR

## License

MIT
