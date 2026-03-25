# Kubernetes Fundamentals

## What Is Kubernetes?

Kubernetes (K8s) orchestrates containers across a cluster of machines. It handles deployment, scaling, networking, and self-healing. You declare the desired state ("run 3 copies of my app") and Kubernetes continuously works to make reality match.

## Core Concepts

| Concept | Purpose | Analogy |
|---------|---------|---------|
| **Pod** | Smallest deployable unit (one or more containers) | A single running instance |
| **Deployment** | Manages a set of identical pods | "Run 3 copies of my app" |
| **Service** | Stable network endpoint for pods | Internal load balancer |
| **Ingress** | External traffic routing (HTTP/HTTPS) | Reverse proxy / external load balancer |
| **ConfigMap** | Non-sensitive configuration | Environment variables file |
| **Secret** | Sensitive configuration | Encrypted env vars |
| **Namespace** | Logical cluster partitioning | Folders for resources |

### Pods

A Pod is one or more containers that share networking and storage. In practice, most pods contain a single container. Multi-container pods are used for sidecar patterns (logging agent, service mesh proxy).

Pods are ephemeral. They can be killed and recreated at any time. Never treat a pod as a pet -- it is cattle.

### Deployments

A Deployment manages a ReplicaSet, which manages Pods. You specify the desired number of replicas and the pod template. Kubernetes ensures that many pods are running at all times.

When you update the pod template (e.g., new image version), Kubernetes performs a rolling update by default.

### Services

Pods get new IP addresses every time they restart. A Service provides a stable DNS name and IP that routes to healthy pods matching a label selector.

Types:
- **ClusterIP** (default) -- internal-only, reachable within the cluster.
- **NodePort** -- exposes the service on a port on every node.
- **LoadBalancer** -- provisions a cloud load balancer (AWS ELB, GCP LB).

### Ingress

Ingress routes external HTTP/HTTPS traffic to Services based on hostnames and paths. It requires an Ingress Controller (e.g., nginx-ingress, Traefik, AWS ALB Ingress Controller).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080
```

### ConfigMaps and Secrets

ConfigMaps hold non-sensitive configuration (feature flags, URLs). Secrets hold sensitive data (API keys, database passwords). Both can be injected as environment variables or mounted as files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  DATABASE_URL: "postgres://db.internal:5432/myapp"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: "hunter2"
```

Important: Kubernetes Secrets are base64-encoded, not encrypted. Use external secret managers (HashiCorp Vault, AWS Secrets Manager) or sealed-secrets for real security.

## Example Deployment

A complete Kubernetes deployment for a Rust service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:v1.2.3
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
```

### Resource requests vs limits

- **Requests:** the minimum resources guaranteed to the pod. The scheduler uses this to place pods on nodes.
- **Limits:** the maximum resources the pod can use. Exceeding memory limits triggers OOM kill. Exceeding CPU limits triggers throttling.

Set requests based on normal usage and limits based on peak usage. Generous limits with tight requests give headroom without waste.

## Health Checks in Rust (axum)

Kubernetes uses health probes to determine pod state:

- **Liveness probe:** "Is the process alive?" Failure triggers a pod restart.
- **Readiness probe:** "Should the load balancer send traffic here?" Failure removes the pod from the Service.

```text
STRUCTURE AppState:
    ready ← shared read-write boolean

STRUCTURE HealthResponse:
    status ← string
    version ← string

/// Liveness: returns 200 as long as the server can respond.
PROCEDURE LIVENESS():
    RETURN JSON { status ← "alive", version ← PACKAGE_VERSION }

/// Readiness: returns 503 during shutdown so the load balancer
/// drains traffic before the pod terminates.
PROCEDURE READINESS(state):
    ready ← AWAIT READ state.ready
    IF ready THEN
        code ← 200 OK
    ELSE
        code ← 503 SERVICE UNAVAILABLE

    RETURN (code, JSON {
        status ← IF ready THEN "ready" ELSE "shutting_down",
        version ← PACKAGE_VERSION
    })
```

## Graceful Shutdown in Rust (axum + tokio)

During a rolling deployment, Kubernetes sends `SIGTERM` to the old pod. The pod must finish in-flight requests before exiting. Without graceful shutdown, users see failed requests.

```text
PROCEDURE MAIN():
    state ← AppState { ready ← shared(true) }

    app ← NEW Router
        REGISTER "/healthz" with GET → LIVENESS
        REGISTER "/ready" with GET → READINESS
        // ... application routes ...
        SET state

    listener ← AWAIT BIND("0.0.0.0:8080")

    AWAIT SERVE(listener, app)
        WITH graceful shutdown ← SHUTDOWN_SIGNAL(state)

PROCEDURE SHUTDOWN_SIGNAL(state):
    WAIT FOR EITHER:
        - CTRL+C signal
        - SIGTERM signal

    // Mark not-ready so readiness probe fails and Kubernetes
    // stops sending new traffic to this pod.
    AWAIT WRITE state.ready ← false

    // Give the load balancer time to notice the failing probe.
    SLEEP 5 seconds
```

**Shutdown sequence during a rolling deployment:**

1. Kubernetes sends `SIGTERM` to the pod.
2. `shutdown_signal` fires, sets `ready` to `false`.
3. Readiness probe returns 503; the Service stops routing new requests.
4. After 5-second grace period, axum stops accepting new connections but finishes in-flight requests.
5. Server exits cleanly. Kubernetes removes the pod.

## When Kubernetes Is Overkill

Kubernetes is powerful but complex. It adds operational burden: cluster upgrades, networking (CNI), RBAC, monitoring the monitoring, etcd maintenance.

**Use Kubernetes when:**
- Running 5+ services with different scaling needs.
- You need auto-scaling, self-healing, and rolling updates.
- Your team has (or is willing to invest in) Kubernetes operational expertise.
- You are already on a managed K8s service (EKS, GKE, AKS) that handles control plane.

**Avoid Kubernetes when:**
- You have a single service or a small number of services.
- Your team lacks Kubernetes experience and cannot invest in learning it.
- A PaaS (Fly.io, Railway, Render) or serverless (AWS Lambda, CloudFlare Workers) is sufficient.
- You are a small team where operational simplicity matters more than orchestration features.

The honest test: if you are spending more time managing Kubernetes than building your product, it is overkill.

## Common Mistakes

- **No resource limits.** A single pod can consume all node resources, starving other pods. Always set requests and limits.
- **Using `latest` image tag.** Deployments cannot track what version is running. Tag with Git SHA or semantic version.
- **Secrets in plain ConfigMaps.** ConfigMaps are stored unencrypted in etcd. Use Secrets (and external secret managers for real security).
- **No readiness probes.** Traffic routes to pods before they are ready, causing errors during startup and deployment.
- **Ignoring pod disruption budgets.** Without a PDB, Kubernetes can kill all your pods during a node drain.

## Key Takeaways

1. Kubernetes manages the lifecycle of containerized applications: deployment, scaling, networking, self-healing.
2. Pods are ephemeral. Deployments manage replicas. Services provide stable endpoints. Ingress routes external traffic.
3. Health checks (liveness + readiness probes) are essential for zero-downtime deployments.
4. Graceful shutdown prevents failed requests during rolling updates.
5. Kubernetes is not always the answer. Evaluate whether a simpler platform meets your needs before adopting it.
