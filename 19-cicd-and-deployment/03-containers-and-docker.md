# Containers & Docker

## What Are Containers?

Containers package an application with all its dependencies into a portable, isolated unit that runs consistently across environments. Unlike virtual machines, containers share the host kernel, making them lightweight and fast to start.

```
Virtual Machine:                    Container:
┌─────────────────┐                ┌─────────────────┐
│   Application   │                │   Application   │
├─────────────────┤                ├─────────────────┤
│   Libraries     │                │   Libraries     │
├─────────────────┤                ├─────────────────┤
│   Guest OS      │                │  (no guest OS)  │
├─────────────────┤                └────────┬────────┘
│   Hypervisor    │                         │
├─────────────────┤                ┌────────┴────────┐
│   Host OS       │                │   Host OS       │
└─────────────────┘                └─────────────────┘
```

A VM includes an entire guest operating system (gigabytes). A container shares the host kernel and only packages the application and its direct dependencies (megabytes).

## Docker Internals

Docker uses three Linux kernel features to isolate containers:

### Namespaces

Namespaces give each container its own isolated view of system resources:

| Namespace | Isolates | Effect |
|-----------|----------|--------|
| **PID** | Process IDs | Container sees only its own processes; PID 1 is the container's entrypoint |
| **NET** | Network stack | Container gets its own IP address, ports, routing table |
| **MNT** | Filesystem mounts | Container has its own root filesystem |
| **UTS** | Hostname | Container can have its own hostname |
| **IPC** | Inter-process communication | Shared memory and semaphores are isolated |
| **USER** | User/group IDs | Root inside container can map to non-root on host |

### cgroups (Control Groups)

cgroups limit and account for resource usage:

- **CPU** -- limit a container to N cores or a percentage of CPU time.
- **Memory** -- set a hard memory limit; the kernel OOM-kills the container if exceeded.
- **I/O** -- throttle disk read/write bandwidth.
- **PIDs** -- limit the number of processes (prevents fork bombs).

These limits are what Kubernetes `resources.requests` and `resources.limits` configure under the hood.

### Union Filesystem (OverlayFS)

Docker images are built in layers. Each Dockerfile instruction creates a new layer. Layers are read-only and shared between images.

```
Layer 4: COPY target/release/myapp    (application binary, ~10 MB)
Layer 3: RUN apt-get install ...       (runtime deps, ~30 MB)
Layer 2: debian:bookworm-slim          (minimal OS, ~80 MB)
Layer 1: (base)
```

When a container runs, a thin read-write layer is added on top. This is why containers start almost instantly -- they do not copy the image; they overlay a writable layer on shared read-only layers.

## Multi-Stage Dockerfile for Rust

The Rust toolchain is large (~1.5 GB). You do not want it in your production image. Multi-stage builds solve this by using one stage to compile and another to package only the binary.

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────────
FROM rust:1.77 AS builder
WORKDIR /app

# Copy manifests first for dependency caching.
COPY Cargo.toml Cargo.lock ./

# Create a dummy main.rs to build dependencies only.
# This layer is cached unless Cargo.toml or Cargo.lock change.
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src

# Now copy real source and build.
COPY src ./src
RUN cargo build --release

# ── Stage 2: Runtime (minimal image) ───────────────────────────
FROM debian:bookworm-slim
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Run as non-root user.
RUN useradd --create-home appuser
USER appuser

COPY --from=builder /app/target/release/myapp /usr/local/bin/
EXPOSE 8080
CMD ["myapp"]
```

### Why multi-stage matters

| | Build image | Runtime image |
|---|---|---|
| **Size** | ~1.5 GB (Rust toolchain + deps) | ~90 MB (binary + minimal OS) |
| **Attack surface** | Compiler, build tools, source code | Only the binary and CA certs |
| **Deploy speed** | Slow to pull | Fast to pull |

### Dependency caching trick

The "dummy main.rs" step is important. Docker caches layers by content hash. If you copy `src/` before building dependencies, any source change invalidates the dependency cache and forces a full rebuild. By building dependencies in a separate step, they are only rebuilt when `Cargo.toml` or `Cargo.lock` changes.

## Image Optimization

### Size reduction techniques

1. **Use slim or distroless base images.** `debian:bookworm-slim` is ~80 MB. `gcr.io/distroless/cc-debian12` is ~20 MB. Alpine is ~5 MB but uses musl libc (can cause issues with some Rust crates).
2. **Multi-stage builds.** Covered above. Never ship the compiler.
3. **Minimize layers.** Combine `RUN` commands where logical to reduce layer count.
4. **Remove package manager caches.** Always add `&& rm -rf /var/lib/apt/lists/*` after `apt-get install`.
5. **Use `.dockerignore`.** Exclude `target/`, `.git/`, test data, and documentation from the build context.

Example `.dockerignore`:
```
target/
.git/
.github/
*.md
tests/
benches/
```

### Static linking for minimal images

For the smallest possible image, compile a fully static binary and use `scratch` (empty) or `distroless`:

```dockerfile
FROM rust:1.77 AS builder
RUN rustup target add x86_64-unknown-linux-musl
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

This produces an image that contains only your binary -- typically under 20 MB. The downside: no shell for debugging, no package manager, no CA certificates (bundle them in or use `distroless`).

## Container Security

### Principle of least privilege

- **Run as non-root.** Add `USER appuser` in your Dockerfile. Never run production containers as root.
- **Read-only filesystem.** Use `--read-only` flag or Kubernetes `readOnlyRootFilesystem: true`. Write only to explicitly mounted volumes.
- **Drop capabilities.** Containers inherit Linux capabilities by default. Drop all and add back only what is needed.

### Image scanning

Scan images for known vulnerabilities before deploying:

- **Trivy** (`trivy image myapp:latest`) -- fast, open-source scanner.
- **Grype** (`grype myapp:latest`) -- alternative from Anchore.
- **GitHub Dependabot / Container Scanning** -- integrated into CI.

Run scans in CI so vulnerable images never reach production:

```yaml
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    severity: CRITICAL,HIGH
    exit-code: 1
```

### Image signing

Sign images to verify they came from your CI pipeline and were not tampered with:

- **cosign** (from Sigstore) signs and verifies container images.
- Kubernetes admission controllers (e.g., Kyverno, OPA Gatekeeper) can enforce that only signed images are deployed.

### Base image hygiene

- Pin base image versions: `FROM debian:bookworm-slim@sha256:abc123...` not `FROM debian:latest`.
- Rebuild images regularly to pick up security patches in base layers.
- Use `docker scout` or `trivy` to monitor base image CVEs.

## Common Mistakes

- **Using `latest` tag.** `latest` is mutable -- you never know what version you are running. Tag with the Git SHA: `myapp:a1b2c3d`.
- **Running as root.** Default Docker containers run as root. A container escape with root privileges compromises the host.
- **Large images.** Shipping the entire build toolchain. Fix: multi-stage builds.
- **No `.dockerignore`.** Sending the entire `.git/` directory and `target/` folder to the Docker daemon. Slows builds and leaks information.
- **"It works on my machine" Docker.** Dockerfile depends on host state (cached layers, local files not in the build context). Fix: build from a clean CI environment.

## Key Takeaways

1. Containers solve "works on my machine" by packaging the application with its dependencies.
2. Docker uses namespaces (isolation), cgroups (resource limits), and union filesystems (efficient layering).
3. Multi-stage builds are essential for Rust: build in a full toolchain image, run in a minimal runtime image.
4. Image size matters: smaller images deploy faster and have less attack surface.
5. Security is not optional: run as non-root, scan for vulnerabilities, sign images, pin base versions.
