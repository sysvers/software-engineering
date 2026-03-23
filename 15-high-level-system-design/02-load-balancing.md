# Load Balancing

## What is Load Balancing?

A load balancer distributes incoming network traffic across multiple backend servers. It is one of the most fundamental building blocks of scalable systems. Without a load balancer, a single server is both a bottleneck and a single point of failure.

```
                     ┌──→ Server 1
Client ──→ [Load    ]├──→ Server 2
            Balancer ├──→ Server 3
                     └──→ Server 4
```

Load balancers serve two primary purposes:
1. **Distribution** — spread traffic evenly so no single server is overwhelmed
2. **Availability** — if a server dies, the load balancer stops sending it traffic and the remaining servers absorb the load

## Layer 4 vs Layer 7 Load Balancing

Load balancers operate at different layers of the network stack, and the choice between them has significant implications.

### Layer 4 (Transport Layer)

L4 load balancers make routing decisions based on TCP/UDP information: source IP, destination IP, source port, and destination port. They never inspect the application payload.

**How it works:** The load balancer receives a TCP connection, selects a backend server, and forwards all packets for that connection to that server. It may use NAT (rewrite the destination IP) or DSR (Direct Server Return, where responses bypass the load balancer).

**Advantages:**
- Extremely fast — no payload inspection, operates at near wire speed
- Protocol-agnostic — works with any TCP/UDP protocol (HTTP, gRPC, database connections, custom protocols)
- Lower resource consumption — can handle millions of concurrent connections

**Use cases:**
- Database connection pooling (MySQL, PostgreSQL)
- Non-HTTP protocols (MQTT for IoT, custom TCP protocols)
- When raw performance matters more than routing flexibility

**Real-world examples:** AWS Network Load Balancer (NLB), HAProxy in TCP mode, Linux IPVS.

### Layer 7 (Application Layer)

L7 load balancers inspect the HTTP (or HTTPS) request and make routing decisions based on application-level information: URL path, HTTP headers, cookies, request body.

**How it works:** The load balancer terminates the client's TCP connection, parses the HTTP request, makes a routing decision, and opens a new connection to the selected backend. This is a full proxy — two separate TCP connections.

**Advantages:**
- Content-based routing — route `/api/users` to the user service and `/api/orders` to the order service
- SSL termination — decrypt HTTPS at the load balancer so backends handle plain HTTP
- Header manipulation — add `X-Forwarded-For`, modify cookies, inject tracing headers
- Request inspection — reject malformed requests before they reach backends
- Compression and caching — the load balancer can compress responses or serve cached content

**Use cases:**
- Microservice routing (path-based or header-based)
- A/B testing (route 10% of traffic to the canary deployment)
- API gateway functionality
- WebSocket connection handling

**Real-world examples:** AWS Application Load Balancer (ALB), NGINX, Envoy, Traefik.

### Comparison Table

| Feature | L4 Load Balancer | L7 Load Balancer |
|---------|-----------------|-----------------|
| **Operates on** | TCP/UDP (IP + port) | HTTP (URLs, headers, cookies) |
| **Routing** | By IP/port only | By URL path, header, cookie, content |
| **Speed** | Faster (no payload parsing) | Slower (parses HTTP) |
| **SSL** | Pass-through or terminate | Typically terminates |
| **Features** | Simple distribution | URL routing, SSL termination, header manipulation |
| **Use case** | Database connections, non-HTTP | Web APIs, microservice routing |
| **Cost** | Lower (less compute) | Higher (more compute per request) |

### Real-World Infrastructure: GitHub's Load Balancing

GitHub uses a multi-layer approach. An L4 load balancer (GLB, their custom solution built on IPVS) sits at the edge, handling millions of TCP connections with minimal overhead. Behind it, L7 routing (NGINX) directs requests to the appropriate backend service based on the URL path. This layered approach gets the performance of L4 at the edge with the flexibility of L7 for service routing.

## Load Balancing Algorithms

The algorithm determines which backend server receives each request.

### Round Robin

Each request goes to the next server in order: Server 1, Server 2, Server 3, Server 1, Server 2, ...

- **Best for:** Equal-capacity servers handling stateless requests of similar cost
- **Weakness:** Does not account for server load. If one request takes 100ms and another takes 10s, servers become unevenly loaded

### Weighted Round Robin

Like round robin, but servers with higher capacity receive proportionally more requests. If Server A has weight 3 and Server B has weight 1, A gets 3 requests for every 1 that B gets.

- **Best for:** Mixed server capacities (e.g., migrating to new hardware while old servers are still in the pool)
- **Weakness:** Weights are static — they do not adapt to real-time load

### Least Connections

Each request goes to the server with the fewest active connections.

- **Best for:** Requests with variable duration (some fast, some slow). Naturally balances load because slow servers accumulate connections and stop getting new ones
- **Weakness:** Does not account for server capacity (a small server with 5 connections may be more loaded than a large server with 10)

### Weighted Least Connections

Combines least connections with server weights. The server with the lowest ratio of (active connections / weight) gets the next request.

- **Best for:** Mixed capacities with variable request duration — the most generally applicable algorithm

### IP Hash

The client's IP address is hashed to determine the server. The same client always goes to the same server.

- **Best for:** Session affinity without sticky sessions or shared session storage
- **Weakness:** Uneven distribution if traffic comes from a few large NATs (corporate networks). Server additions/removals cause widespread reassignment (mitigated by consistent hashing)

### Random

Pick a server at random for each request.

- **Best for:** Surprisingly effective at scale due to the law of large numbers. Simple to implement and avoids the state tracking of least-connections
- **Weakness:** Can create imbalances with small server counts

### Power of Two Random Choices

Pick two servers at random, send the request to the one with fewer connections.

- **Best for:** Large server pools. Provides near-optimal distribution with minimal coordination. Used by Envoy proxy
- **Why it works:** Choosing the better of two random options avoids the "herd" problem (all clients simultaneously picking the same "least loaded" server)

## Health Checks

A load balancer must know which backend servers are healthy. It does this through health checks — periodic probes sent to each server.

### Types of Health Checks

**Passive health checks:** The load balancer monitors real traffic. If a server returns 5xx errors or connections time out, it is marked unhealthy. No extra probes needed, but detection is slower.

**Active health checks:** The load balancer sends periodic requests to a dedicated health endpoint. Faster detection but adds load to backends.

**Best practice:** Use both. Active health checks catch servers that are alive but broken (e.g., can connect but the application has deadlocked). Passive checks catch issues between probe intervals.

### Health Check Configuration

Typical parameters:
- **Interval:** How often to check (e.g., every 5 seconds)
- **Timeout:** How long to wait for a response (e.g., 3 seconds)
- **Unhealthy threshold:** How many consecutive failures before marking unhealthy (e.g., 3)
- **Healthy threshold:** How many consecutive successes before marking healthy again (e.g., 2)

The unhealthy threshold prevents a single dropped packet from removing a healthy server. The healthy threshold prevents a flapping server from being re-added too quickly.

### Rust Health Check Endpoint

A production health check endpoint should report meaningful status. Here is a comprehensive example:

```rust
use axum::{routing::get, Json, Router};
use serde::Serialize;
use std::sync::Arc;
use std::time::Instant;
use tokio::net::TcpListener;

#[derive(Serialize)]
struct HealthResponse {
    status: &'static str,
    uptime_secs: u64,
    version: &'static str,
    checks: HealthChecks,
}

#[derive(Serialize)]
struct HealthChecks {
    database: CheckResult,
    cache: CheckResult,
}

#[derive(Serialize)]
struct CheckResult {
    status: &'static str,
    latency_ms: u64,
}

static START_TIME: std::sync::OnceLock<Instant> = std::sync::OnceLock::new();

/// Deep health check: verifies all dependencies.
/// The load balancer calls this endpoint. If it returns non-200
/// or times out, the server is removed from the pool.
async fn health_check(state: Arc<AppState>) -> Json<HealthResponse> {
    let start = START_TIME.get_or_init(Instant::now);

    // Check database connectivity
    let db_start = Instant::now();
    let db_ok = sqlx::query("SELECT 1")
        .execute(&state.db_pool)
        .await
        .is_ok();
    let db_latency = db_start.elapsed().as_millis() as u64;

    // Check cache connectivity
    let cache_start = Instant::now();
    let cache_ok = state.redis.ping().await.is_ok();
    let cache_latency = cache_start.elapsed().as_millis() as u64;

    let all_healthy = db_ok && cache_ok;

    Json(HealthResponse {
        status: if all_healthy { "healthy" } else { "degraded" },
        uptime_secs: start.elapsed().as_secs(),
        version: env!("CARGO_PKG_VERSION"),
        checks: HealthChecks {
            database: CheckResult {
                status: if db_ok { "up" } else { "down" },
                latency_ms: db_latency,
            },
            cache: CheckResult {
                status: if cache_ok { "up" } else { "down" },
                latency_ms: cache_latency,
            },
        },
    })
}

/// Shallow liveness check: just confirms the process is running.
/// Use this for Kubernetes liveness probes (restart if dead).
/// Use the deep health check for readiness probes (stop sending traffic).
async fn liveness() -> &'static str {
    "ok"
}

struct AppState {
    db_pool: sqlx::PgPool,
    redis: redis::Client,
}

#[tokio::main]
async fn main() {
    START_TIME.get_or_init(Instant::now);

    let state = Arc::new(AppState {
        db_pool: sqlx::PgPool::connect("postgres://localhost/mydb")
            .await
            .unwrap(),
        redis: redis::Client::open("redis://localhost").unwrap(),
    });

    let app = Router::new()
        .route("/health", get({
            let state = state.clone();
            move || health_check(state.clone())
        }))
        .route("/livez", get(liveness));

    let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**Key design decisions:**
- Separate liveness (`/livez`) from readiness (`/health`). Liveness answers "is the process alive?" — readiness answers "can it serve traffic?"
- The deep health check tests actual dependencies (database, cache). A server that cannot reach its database should not receive traffic
- Include latency in the response for monitoring dashboards

## Session Affinity (Sticky Sessions)

Some applications store session state on the server (e.g., shopping cart in memory, authentication tokens in local storage). Session affinity ensures a user's requests always go to the same server.

### Approaches

**Cookie-based:** The load balancer sets a cookie with the server ID. Subsequent requests from the same client include the cookie, and the load balancer routes accordingly.

**IP-based:** Use the client's IP address to determine the server (IP hash algorithm). Simpler but breaks when clients are behind NAT or change networks.

**Header-based:** Route based on a custom header (e.g., `X-Session-Server`). Useful for API clients.

### Why Sticky Sessions Are Usually a Bad Idea

- **Uneven load distribution** — a "sticky" user making many requests overloads one server while others idle
- **Failover breaks sessions** — if the server dies, the user's session is lost
- **Scaling is harder** — adding a new server does not redistribute existing sessions

**The better approach:** Store session state externally (Redis, database). This makes your application servers stateless, allowing the load balancer to route freely using least-connections or round-robin. Stateless servers are easier to scale, easier to replace, and easier to deploy.

### Real-World Example: Shopify's Approach

Shopify serves millions of merchants with stateless application servers. All session state lives in Redis. When a server dies, the next server reads the session from Redis and continues seamlessly. This lets them deploy new code to any server independently and scale horizontally without worrying about session pinning.

## Advanced Topics

### Global Server Load Balancing (GSLB)

For multi-region deployments, DNS-based load balancing routes users to the nearest region. AWS Route 53, Cloudflare, and Google Cloud DNS support latency-based or geolocation-based routing.

```
User in Tokyo  ──→ DNS ──→ Tokyo datacenter
User in London ──→ DNS ──→ London datacenter
User in NYC    ──→ DNS ──→ Virginia datacenter
```

### Connection Draining

When removing a server from the pool (for deployment or maintenance), do not kill active connections immediately. Connection draining stops sending new requests to the server while allowing in-flight requests to complete (with a timeout). This prevents user-visible errors during deployments.

### Load Balancer as a Single Point of Failure

The load balancer itself can be a SPOF. Solutions:
- **Active-passive pair:** Two load balancers share a virtual IP (VIP). If the primary fails, the secondary takes over via VRRP or similar protocol
- **Active-active pair:** Both load balancers serve traffic simultaneously, with DNS round-robin or anycast distributing between them
- **Cloud-managed:** AWS ALB/NLB, Google Cloud Load Balancing, and Azure Load Balancer are managed services with built-in redundancy

## Common Pitfalls

- **Using L7 when L4 suffices.** If you do not need content-based routing, L4 is faster and cheaper. Do not pay the overhead of HTTP parsing for database connections.
- **No health checks.** Without health checks, the load balancer happily sends traffic to dead servers. Always configure both active and passive checks.
- **Sticky sessions masking a stateful design.** If your application requires sticky sessions, that is a design smell. Fix the root cause by externalizing state.
- **Ignoring connection limits.** Each backend server has a maximum number of concurrent connections. If the load balancer sends more, connections queue or fail. Set backend connection limits in the load balancer configuration.
- **Not testing failover.** Regularly test what happens when a backend server dies. Does the load balancer detect it quickly? Does traffic redistribute smoothly?

## Key Takeaways

1. Use L4 for non-HTTP protocols and raw performance; use L7 for content-based routing and HTTP features.
2. Least connections (or weighted least connections) is the safest default algorithm for most workloads.
3. Health checks are non-negotiable — both active and passive.
4. Avoid sticky sessions; externalize state instead.
5. The load balancer itself needs redundancy — do not create a new single point of failure.
