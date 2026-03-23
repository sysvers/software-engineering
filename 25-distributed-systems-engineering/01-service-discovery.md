# Service Discovery

## Overview

In a distributed system composed of many services, each service needs to locate the others. Hardcoding addresses breaks down when services scale horizontally, move across hosts, or restart on different ports. Service discovery solves this by maintaining a registry of available service instances and their network locations.

---

## Discovery Models

### Client-Side Discovery

The client queries a service registry and selects an instance using a load-balancing strategy. The client bears the responsibility of choosing a healthy instance.

**Flow:**

```
Client --> Service Registry --> [Instance A, Instance B, Instance C]
Client picks Instance B (round-robin, random, least-connections, etc.)
Client --> Instance B
```

**Advantages:**
- No extra network hop through a load balancer
- Client can implement sophisticated selection logic (latency-based, zone-aware)
- Fewer infrastructure components to manage

**Disadvantages:**
- Every client language/framework must implement discovery logic
- Tight coupling between clients and the registry
- Harder to enforce centralized routing policies

### Server-Side Discovery

The client sends requests to a load balancer or API gateway, which queries the registry and forwards the request. This simplifies the client at the cost of an additional network hop.

**Flow:**

```
Client --> Load Balancer / API Gateway --> Service Registry
                                      --> Instance B
```

**Advantages:**
- Client is simple -- just talk to one endpoint
- Centralized place to enforce policies (rate limiting, auth, canary routing)
- Language-agnostic: any client that speaks HTTP/gRPC works

**Disadvantages:**
- Extra network hop adds latency
- Load balancer is a potential bottleneck or single point of failure
- More infrastructure to deploy and monitor

---

## DNS-Based Discovery

The simplest form of service discovery uses DNS. Services register themselves (or are registered by a scheduler) as DNS records. Clients resolve the service name to one or more IP addresses.

**Plain DNS** (e.g., SRV records) works but has caching issues -- DNS TTLs mean stale entries linger after a node goes down. Most modern systems layer health checking on top of DNS to mitigate this.

**Key limitation:** DNS does not carry metadata (version, weight, health status) natively. Systems like Consul DNS bridge this gap by serving DNS responses backed by a health-checked registry.

---

## Service Registries

### Consul (HashiCorp)

Consul provides service discovery, health checking, KV storage, and multi-datacenter support. Services register via an agent running on each node. Health checks (HTTP, TCP, script, or TTL-based) automatically remove unhealthy instances from query results.

- **Gossip protocol** (Serf) for membership and failure detection
- **Raft consensus** for strongly consistent KV store and leader election
- **DNS and HTTP interfaces** for discovery queries
- **Service mesh** via Consul Connect with mutual TLS

### etcd

etcd is a distributed KV store using Raft consensus. While not a service registry by itself, it is commonly used as the backing store for service discovery. Kubernetes uses etcd as its source of truth for all cluster state, including service endpoints.

- **Strong consistency** via Raft
- **Watch API** for real-time notifications of changes
- **Lease-based TTLs** for ephemeral registrations

### ZooKeeper

An older but battle-tested coordination service. Used by Kafka (pre-KRaft), Hadoop, and HBase. Provides ephemeral nodes that automatically disappear when a client session ends, which maps naturally to service instance lifecycle.

---

## Rust Implementation

A simple in-memory service registry with TTL-based health tracking:

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use std::time::{Duration, Instant};

/// Represents a single service instance in the registry.
#[derive(Debug, Clone)]
struct ServiceInstance {
    id: String,
    host: String,
    port: u16,
    last_heartbeat: Instant,
}

/// A simple in-memory service registry with TTL-based health tracking.
struct ServiceRegistry {
    services: Arc<RwLock<HashMap<String, Vec<ServiceInstance>>>>,
    ttl: Duration,
}

impl ServiceRegistry {
    fn new(ttl: Duration) -> Self {
        Self {
            services: Arc::new(RwLock::new(HashMap::new())),
            ttl,
        }
    }

    /// Register a service instance under a given service name.
    fn register(&self, service_name: &str, instance: ServiceInstance) {
        let mut map = self.services.write().unwrap();
        map.entry(service_name.to_string())
            .or_insert_with(Vec::new)
            .push(instance);
    }

    /// Discover healthy instances for a given service name.
    fn discover(&self, service_name: &str) -> Vec<ServiceInstance> {
        let map = self.services.read().unwrap();
        let now = Instant::now();
        map.get(service_name)
            .map(|instances| {
                instances
                    .iter()
                    .filter(|inst| now.duration_since(inst.last_heartbeat) < self.ttl)
                    .cloned()
                    .collect()
            })
            .unwrap_or_default()
    }

    /// Remove stale instances that have not sent a heartbeat within the TTL.
    fn evict_stale(&self) {
        let mut map = self.services.write().unwrap();
        let now = Instant::now();
        for instances in map.values_mut() {
            instances.retain(|inst| now.duration_since(inst.last_heartbeat) < self.ttl);
        }
    }
}
```

---

## Real-World Examples

### Netflix Eureka

Netflix built Eureka as a REST-based service registry for their AWS-deployed microservices. Key design decisions:

- **AP system** (availability over consistency) -- during a network partition, Eureka preserves existing registrations rather than purging them, preferring stale data over no data
- **Self-preservation mode** -- if too many instances fail heartbeats simultaneously (suggesting a network issue, not mass failure), Eureka stops evicting instances
- **Client-side caching** -- Eureka clients cache the registry locally, so they can still discover services even if the Eureka server is temporarily unavailable
- **Peer replication** -- Eureka servers replicate registrations to each other asynchronously for redundancy

Netflix eventually moved much of their infrastructure to use internal service mesh approaches, but Eureka's design principles remain influential.

### Kubernetes DNS and Service Discovery

Kubernetes provides built-in server-side discovery:

- Every `Service` object gets a stable DNS name (e.g., `my-service.my-namespace.svc.cluster.local`)
- `kube-proxy` or eBPF-based alternatives (Cilium) handle load balancing at the network level
- **Endpoints** objects track the set of healthy pod IPs backing each service
- **Headless services** (no ClusterIP) return individual pod IPs via DNS, enabling client-side load balancing when needed
- etcd stores all service and endpoint state; the control plane watches for pod lifecycle changes and updates endpoints automatically

The Kubernetes model is server-side discovery by default (clients hit a virtual IP, kube-proxy routes to a pod) but supports client-side patterns via headless services and libraries like `kube-rs` in Rust.

---

## Common Pitfalls

1. **Stale registrations**: If health checks are too infrequent or TTLs too long, clients route traffic to dead instances. Use aggressive TTLs (5-15 seconds) with heartbeat intervals at 1/3 of the TTL.

2. **Thundering herd on registry**: If all clients poll the registry simultaneously, it creates load spikes. Use jittered polling intervals and client-side caching.

3. **No fallback on registry failure**: If the registry itself is down, clients must have a cached copy of known instances. Never make the registry a hard synchronous dependency on the request path.

4. **Ignoring zone/region awareness**: Discovering an instance in a distant availability zone adds unnecessary latency. Prefer same-zone instances when possible.

---

## Key Takeaways

- Client-side discovery gives more control; server-side discovery gives simplicity. Most production systems use server-side (Kubernetes, API gateways) because operational simplicity wins at scale.
- DNS-based discovery is the lowest-common-denominator approach but lacks health awareness without additional tooling.
- Consul and etcd are the dominant registry backends in modern systems. ZooKeeper is legacy but still widely deployed.
- Always cache registry results on the client side. The registry is a convenience, not a hard dependency for every request.
