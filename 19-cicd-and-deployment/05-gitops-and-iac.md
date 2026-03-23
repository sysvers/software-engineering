# GitOps & Infrastructure as Code

## GitOps

GitOps uses Git as the single source of truth for infrastructure and application configuration. Changes to infrastructure happen through Git commits and pull requests, not through manual `kubectl apply` or console clicks.

### Core Principle

The state of your cluster should always match what is declared in Git. If someone manually changes something in the cluster, the GitOps agent detects the drift and reverts it.

```
Desired state (Git) ──→ GitOps agent ──→ Actual state (Cluster)
                              ↑
                    Continuously reconciles
```

### GitOps Workflow

```
Developer changes config → PR → Code review → Merge → GitOps agent detects change → Apply to cluster
```

1. A developer modifies a Kubernetes manifest (e.g., updates the image tag from `v1.2.3` to `v1.3.0`).
2. They open a pull request. The team reviews the change.
3. On merge, the GitOps agent (running inside the cluster) detects the new commit.
4. The agent applies the changes to the cluster, bringing actual state in line with desired state.
5. The agent continuously monitors for drift. If someone manually edits the cluster, the agent reverts it.

### ArgoCD

ArgoCD is the most widely adopted GitOps controller for Kubernetes. It provides:

- **Declarative configuration.** Point ArgoCD at a Git repo and a target cluster. It syncs automatically.
- **Web UI.** Visual diff of desired vs actual state, sync status, health status.
- **Automated sync.** Detect changes in Git and apply them without manual intervention.
- **Rollback.** Revert to any previous Git commit -- the cluster state follows.

Example ArgoCD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Revert manual cluster changes
    syncOptions:
      - CreateNamespace=true
```

### Flux

Flux is an alternative GitOps toolkit, now a CNCF graduated project. It takes a more modular approach:

- **Source Controller** watches Git repos and Helm repos.
- **Kustomize Controller** applies Kustomize overlays.
- **Helm Controller** manages Helm releases.
- **Notification Controller** sends alerts on sync events.

Flux is lighter-weight than ArgoCD (no UI by default) and integrates well with Kustomize-heavy workflows.

### ArgoCD vs Flux

| Feature | ArgoCD | Flux |
|---------|--------|------|
| **UI** | Built-in web dashboard | No built-in UI (third-party options exist) |
| **Architecture** | Monolithic application | Modular controllers |
| **Multi-cluster** | Native support | Supported via remote clusters |
| **Helm support** | Yes | Yes (via Helm Controller) |
| **Best for** | Teams wanting visibility and a UI | Teams preferring CLI-first, GitOps-native |

### Benefits of GitOps

- **Audit trail.** Git history records who changed what, when, and why. Every infrastructure change has a commit message and a PR review.
- **Rollback.** Revert a commit and the cluster follows. No need to remember what manual commands were run.
- **Consistency.** Declared state equals actual state. No configuration drift.
- **Review process.** Infrastructure changes go through pull requests, just like code.
- **Disaster recovery.** If the cluster is lost, re-create it and point the GitOps agent at the same repo. The entire state is reconstructed.

## Infrastructure as Code (IaC)

IaC manages infrastructure (servers, databases, networks, DNS, load balancers) through declarative configuration files rather than manual console operations.

### Why IaC?

- **Reproducibility.** Create identical environments (staging, production) from the same code.
- **Version control.** Track infrastructure changes with the same Git workflow as application code.
- **Automation.** No manual clicking through cloud consoles. `terraform apply` creates everything.
- **Documentation.** The code *is* the documentation of what infrastructure exists.

### Terraform

Terraform is the most widely used IaC tool. It uses HCL (HashiCorp Configuration Language) to declare resources.

```hcl
resource "aws_rds_instance" "main" {
  identifier        = "myapp-production"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.r6g.large"
  allocated_storage = 100

  db_name  = "myapp"
  username = var.db_username
  password = var.db_password

  multi_az               = true
  backup_retention_period = 7

  tags = {
    Environment = "production"
  }
}
```

**Terraform workflow:**

```
terraform init     → Download providers and initialize state
terraform plan     → Preview what will change (dry run)
terraform apply    → Apply the changes
terraform destroy  → Tear down all resources
```

The `plan` step is critical: it shows exactly what will be created, modified, or destroyed before you commit to the change.

### Pulumi

Pulumi takes a different approach: use real programming languages (TypeScript, Python, Go, Rust) instead of a domain-specific language.

```typescript
import * as aws from "@pulumi/aws";

const db = new aws.rds.Instance("main", {
    identifier: "myapp-production",
    engine: "postgres",
    engineVersion: "15.4",
    instanceClass: "db.r6g.large",
    allocatedStorage: 100,
    dbName: "myapp",
    username: dbUsername,
    password: dbPassword,
    multiAz: true,
    backupRetentionPeriod: 7,
    tags: { Environment: "production" },
});
```

### Terraform vs Pulumi

| Feature | Terraform | Pulumi |
|---------|-----------|--------|
| **Language** | HCL (domain-specific) | Real programming languages (TypeScript, Python, Go, etc.) |
| **State** | Remote state file (S3, Terraform Cloud) | Managed service or self-hosted |
| **Ecosystem** | Largest provider ecosystem | Growing, can use Terraform providers via bridge |
| **Testing** | Limited (terratest, external tools) | Standard testing frameworks (Jest, pytest, etc.) |
| **Learning curve** | Learn HCL | Use a language you already know |
| **Complex logic** | Awkward (HCL is not a general-purpose language) | Natural (loops, conditionals, abstractions) |
| **Best for** | Most infrastructure, large community | Complex logic, teams with existing language expertise |

For most teams, Terraform is the pragmatic default due to its ecosystem and community. Pulumi shines when infrastructure requires complex logic or when the team strongly prefers a general-purpose language.

## Reproducible Builds

A reproducible build produces identical output from identical input, regardless of when or where it is built.

### Why it matters

- **Security.** Verify that the deployed binary matches the source code. Detect supply-chain tampering.
- **Debugging.** Reproduce the exact build that is running in production.
- **Compliance.** Prove the build chain is untampered for audits and certifications.

### Techniques

**Pin dependency versions.** Use lockfiles (`Cargo.lock` for Rust, `package-lock.json` for Node). Commit them to version control. Without a lockfile, `cargo build` on two different machines may resolve to different dependency versions.

**Pin toolchain versions.** Use `rust-toolchain.toml` to declare the exact Rust version:

```toml
[toolchain]
channel = "1.77.0"
components = ["clippy", "rustfmt"]
```

Every developer and CI machine uses the same compiler version.

**Deterministic build environments.** Use Docker or Nix to ensure the build environment is identical everywhere. No reliance on what happens to be installed on the host.

**Content-addressable artifact storage.** Store build artifacts by their content hash (SHA-256). If two builds produce the same hash, they are identical. This enables verification and deduplication.

## Artifact Management

Build artifacts (Docker images, binaries, packages) need storage, versioning, and distribution.

### Tagging strategy

Tag images with the Git SHA, not just `latest`:

```
myregistry/myapp:a1b2c3d    ← Git commit SHA (immutable, traceable)
myregistry/myapp:v1.3.0     ← Semantic version (for releases)
myregistry/myapp:latest     ← Mutable, avoid for production
```

The Git SHA tag lets you trace any running container back to the exact source code that produced it.

### Registry best practices

- Use a private registry (GitHub Container Registry, AWS ECR, Google Artifact Registry).
- Enable vulnerability scanning on the registry (most managed registries support this).
- Implement image signing with cosign for supply-chain security.
- Set retention policies to automatically clean up old images.
- Use pull-through caches in CI to avoid rate limits on Docker Hub.

### Artifact pipeline

```
Source code → CI build → Artifact (Docker image) → Registry → GitOps agent → Cluster
                  ↓
            Scan for CVEs
                  ↓
            Sign with cosign
```

The artifact is the handoff point between CI and CD. CI produces and validates the artifact. CD (GitOps) deploys it.

## Common Mistakes

- **Manual infrastructure changes.** "I'll just click this in the console real quick." Now your Terraform state and actual state are out of sync. Always make changes through code.
- **No state locking.** Two people running `terraform apply` simultaneously corrupt the state. Use remote state with locking (S3 + DynamoDB, Terraform Cloud).
- **Secrets in Git.** Database passwords, API keys in plain text in Terraform files. Use secret managers (Vault, AWS Secrets Manager) and reference them dynamically.
- **No `terraform plan` review.** Applying without reviewing the plan. A misconfigured resource deletion can destroy production data.
- **Mutable image tags.** Using `latest` or re-tagging an existing version. You lose traceability and reproducibility.

## Key Takeaways

1. GitOps makes infrastructure changes auditable, reviewable, and reversible through Git.
2. ArgoCD and Flux continuously reconcile cluster state with Git. Drift is detected and corrected automatically.
3. Infrastructure as Code (Terraform, Pulumi) makes infrastructure reproducible, version-controlled, and automated.
4. Reproducible builds require pinning dependencies, toolchains, and build environments.
5. Tag artifacts with Git SHAs for traceability. Scan and sign them before deployment.
6. Automate everything. Manual infrastructure changes are where mistakes happen, especially under pressure at 2 AM.
