# Docker Notes

---

### What is Docker?
**Docker is an open-source containerization platform** that lets you package an application together with all its dependencies (libraries, runtime, configuration, system tools) into a single standardized unit called an **image**. That image can then be run as a **container** — an isolated, lightweight, portable process — consistently across any environment that has Docker installed, whether that's your laptop, a teammate's machine, a CI pipeline, or a production server in the cloud.

In one line: *Docker packages "build once, run anywhere" for applications by bundling the app and its entire environment together.*

### Why is Docker Used?
Docker solves problems that plagued software delivery before containerization became mainstream:

- **"Works on my machine" problem** — before Docker, an app could behave differently across dev, test, and prod due to differing OS versions, library versions, or missing dependencies. Docker eliminates this by shipping the exact same environment everywhere.
- **Slow, heavy VMs** — spinning up a full VM per application (with its own OS) is resource-hungry and slow to boot. Containers share the host kernel, so they're lightweight and start in seconds.
- **Complex environment setup** — onboarding a new developer used to mean manually installing a dozen dependencies. With Docker, it's `docker compose up`.
- **Inconsistent deployments** — Docker images are immutable and versioned, so what you tested is exactly what you deploy — no drift.
- **Scaling and orchestration needs** — containers are the standard unit that orchestrators like Kubernetes/EKS schedule and scale, so Docker is the foundational layer beneath your entire DevOps toolchain.

### Uses of Docker
- Packaging microservices so each service runs in its own isolated, consistent environment.
- Local development environments that exactly mirror production (via Compose).
- CI/CD pipelines — building, testing, and shipping the same artifact through every stage.
- Running legacy or conflicting-dependency applications side-by-side on the same host without conflicts.
- Serving as the base unit that Kubernetes/EKS, ECS, and other orchestrators deploy and manage.
- Simplifying "spin up a database/tool locally in one command" for testing (Postgres, Redis, Kafka, etc.) without installing them natively.

### Key Features of Docker
- **Portability** — build once, run identically on any machine with Docker installed (OCI-compliant images run across Docker, containerd, Podman).
- **Lightweight** — containers share the host kernel instead of bundling a full guest OS, unlike VMs.
- **Fast startup** — containers start in seconds since there's no OS boot process.
- **Layered image system** — images are built from reusable, cacheable layers, saving build time and storage.
- **Isolation** — namespaces and cgroups isolate each container's processes, network, and filesystem view, and enforce resource limits.
- **Version control for images** — every image can be tagged and versioned, and layers are diffable, similar to Git for infrastructure.
- **Huge ecosystem** — Docker Hub and other registries offer prebuilt images for almost anything (databases, message queues, web servers).
- **Scalability** — integrates natively with orchestration tools (Kubernetes, ECS, Swarm) for scaling containers horizontally.
- **Consistency across environments** — dev, staging, and prod all run the exact same containerized artifact.

---

## 1. Core Concepts & Architecture

### Container vs VM
A **VM** virtualizes hardware — each VM runs a full guest OS with its own kernel, on top of a hypervisor (ESXi, KVM, Hyper-V).

A **container** virtualizes the OS — all containers on a host share the same kernel, and isolation is enforced via Linux kernel features, not a hypervisor.

Key Linux primitives that make containers possible:
- **Namespaces** — provide isolation. Each container gets its own view of: PID (process IDs), NET (network interfaces), MNT (mount points/filesystem), UTS (hostname), IPC (inter-process communication), USER (user/group IDs).
- **cgroups (control groups)** — enforce resource limits (CPU, memory, I/O) so one container can't starve others.
- **Union filesystem (overlay2)** — lets images be built from stacked, reusable read-only layers with a writable layer on top.

**Why this matters in interviews:** "Containers are lightweight because they share the host kernel" is the one-line answer, but be ready to explain *why* — no guest OS boot, no hypervisor overhead, faster startup (seconds vs minutes), smaller footprint.

### Docker Engine Architecture

**Docker uses a client-server architecture** where the **Docker Client** communicates with the **Docker Daemon (dockerd)**, which handles the heavy lifting of building, running, and distributing containers. They interact over a **REST API** using UNIX sockets or a network interface.

```text
  +------------------+                 +------------------------------------+                 +--------------------+

  |  Docker Client   |                 |            Docker Host             |                 |  Docker Registry   |
  |                  |                 |                                    |                 |                    |
  |  docker build   --------REST API------->  Docker Daemon (dockerd)       |                 |    Docker Hub      |
  |  docker pull     |                 |          |          |              |                 |    (or Private)    |
  |  docker run      |                 |          v          v              |                 |                    |
  +------------------+                 |      Images      Containers        | <---Push/Pull--->                    |

                                       |     (Layers)    (Read/Write)       |                 +--------------------+
                                       +------------------------------------+
```

**The Three Core Pillars**
The overall environment is divided into three major conceptual areas:
- **Docker Client:** The user interface. When you execute commands like docker run, the client sends these requests to the daemon via the Docker Engine REST API.
- **Docker Host:** The physical machine or VM where the Docker daemon runs. It manages local objects like images, containers, networks, and storage.
- **Docker Registry:** A central repository where images are uploaded and downloaded. Docker Hub is the default public option.

**Deep Dive: The Container Runtime Flow**

Modern Docker is highly modular, breaking down the old monolithic engine into open standards (OCI)

- **Docker daemon (`dockerd`)** — background service that manages images, containers, networks, volumes. Listens for API requests.
- **Docker CLI** — the `docker` command you type; talks to the daemon via REST API (usually over a Unix socket `/var/run/docker.sock`).
- **containerd** — a separate daemon that manages the container lifecycle (start/stop/pause) at a lower level. `dockerd` delegates to containerd.
- **runc** — the actual low-level OCI-compliant runtime that creates and runs containers using namespaces/cgroups. containerd calls runc.

So the call chain is: `docker CLI → dockerd → containerd → runc → container process`.

### Images vs Containers vs Registries
- **Image** — a read-only template: application code + dependencies + runtime + config, built in layers.
- **Container** — Live instances. A container adds a thin, writeable **"container layer"** on top of the underlying read-only image layers.
- **Registry** — a storage/distribution service for images (Docker Hub, ECR, GCR, private registries). You `push` to and `pull` from a registry.


### OCI (Open Container Initiative)
A set of open standards so images/runtimes are portable across tools (Docker, Podman, containerd). Two main specs:
- **Image spec** — defines image format/layers/manifest.
- **Runtime spec** — defines how a container should be run.

This is why an image built with Docker can run under containerd or Podman without modification.

### Layers and overlay2
Every Dockerfile instruction that modifies the filesystem creates a new layer. Layers are cached and reused across images — this is the backbone of Docker's speed and storage efficiency. `overlay2` is the default storage driver that stacks these layers.

---

## 2. Images & Dockerfile

### Definition: Docker Image
A **Docker image** is a read-only, immutable template built from a set of stacked filesystem layers that contains an application's code, runtime, libraries, environment variables, and configuration needed to run it. Images are built from a `Dockerfile` and are the blueprint from which containers are created — an image itself never runs; a container (an instance of that image) does.

### Layer Caching
Docker caches each build step. If a layer's instruction and context haven't changed, Docker reuses the cached layer instead of rebuilding it. Once one layer changes, **every layer after it is invalidated** and rebuilt.

**Optimization principle:** order instructions from least-frequently-changed to most-frequently-changed.

```dockerfile
# BAD — code changes invalidate the dependency install layer every time
COPY . .
RUN npm install

# GOOD — dependencies only reinstall when package.json actually changes
COPY package.json package-lock.json ./
RUN npm install
COPY . .
```

### Key Dockerfile Instructions

| Instruction | Purpose |
|---|---|
| `FROM` | Base image to build from |
| `RUN` | Executes a command at **build time**, creates a new layer |
| `CMD` | Default command at **container start**; overridable via `docker run` args |
| `ENTRYPOINT` | Fixed executable for the container; args passed at runtime are appended to it |
| `COPY` | Copies files from build context into the image (simple, predictable) |
| `ADD` | Like COPY but also auto-extracts tar files and can fetch URLs (avoid unless you need that) |
| `WORKDIR` | Sets working directory for subsequent instructions |
| `ENV` | Sets environment variable, persists into the running container |
| `ARG` | Build-time-only variable, not available in the running container |
| `EXPOSE` | Documents which port the container listens on (does NOT actually publish it) |
| `USER` | Sets the user the container process runs as (security best practice: non-root) |
| `HEALTHCHECK` | Defines a command Docker runs periodically to check container health |
| `LABEL` | Metadata key-value pairs on the image |
| `VOLUME` | Declares a mount point as a volume |

**CMD vs ENTRYPOINT — the classic trap:**
- `CMD` is a *default* that's fully replaceable: `docker run myimage echo hi` overrides CMD entirely.
- `ENTRYPOINT` is *fixed*: whatever you pass on `docker run` gets appended as arguments to it.
- Best practice combo: `ENTRYPOINT ["python", "app.py"]` + `CMD ["--verbose"]` — gives a fixed executable with an overridable default flag.

**COPY vs ADD:** Use `COPY` by default — it's explicit and predictable. Only use `ADD` when you specifically need auto-extraction of a local tarball.

**ENV vs ARG:** `ARG` only exists during the build (e.g., choosing a version to install); `ENV` persists into the running container and is visible via `docker inspect` / `printenv`. Never put secrets in either — both end up baked into image layer history.

### Multi-Stage Builds
Lets you use one stage to compile/build (with all the heavy build tools) and copy only the final artifact into a slim runtime stage — dramatically reducing final image size and attack surface.

```dockerfile
# Stage 1: build
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Stage 2: runtime
FROM alpine:3.19
COPY --from=builder /app/myapp /usr/local/bin/myapp
ENTRYPOINT ["myapp"]
```
Final image has no Go compiler, no source code — just the binary. This is a very commonly asked interview topic and directly relevant to your Trivy scanning workflow (smaller image = smaller vulnerability surface).

### .dockerignore
Excludes files from the build context sent to the daemon (e.g., `.git`, `node_modules`, `*.log`). Reduces build context size, speeds up builds, and prevents accidentally copying secrets or bloat into the image.

### Base Image Selection
- **Alpine** — very small (~5MB), musl libc instead of glibc (can cause subtle compatibility issues with some binaries).
- **Slim** (Debian-based, e.g., `python:3.12-slim`) — smaller than full image, glibc-compatible, fewer surprises than Alpine.
- **Distroless** (Google) — no shell, no package manager, just the app and its runtime dependencies. Extremely small attack surface, but harder to debug (no `docker exec sh` into it).
- **Full base images** — easiest to debug, largest size, most vulnerabilities to scan.

Tradeoff: security/size vs debuggability/compatibility.

### Image Inspection
- `docker history <image>` — shows each layer and the command that created it.
- `docker inspect <image>` — full metadata (env vars, entrypoint, exposed ports, layers).

### BuildKit
Modern build engine (default in current Docker versions) offering:
- Parallel execution of independent build stages.
- **Cache mounts** — persist package manager caches (e.g., `apt`, `npm`) across builds without baking them into layers.
- **Secret mounts** — pass secrets into the build (`--mount=type=secret`) without leaving them in the final image or layer history, unlike `ARG`/`ENV`.

---

## 3. Containers — Runtime Behavior

### Definition: Docker Container
A **Docker container** is a running (or stopped) instance of an image — an isolated process with its own filesystem view, network interface, and resource limits, created using Linux namespaces and cgroups, with a thin writable layer on top of the image's read-only layers. 
**Containers** are ephemeral and disposable by design: you can stop, remove, and recreate one from the same image at any time without losing the image itself.

### Container Lifecycle
`docker create` → `docker start` → (running) → `docker stop`/`kill` → `docker rm`. `docker run` = `create` + `start` combined.

- `stop` sends SIGTERM, waits (default 10s grace period), then SIGKILL if not exited.
- `kill` sends SIGKILL (or a specified signal) immediately.
- `pause`/`unpause` freezes/unfreezes all processes in the container via cgroups freezer — useful for point-in-time snapshots without stopping.

### Restart Policies
| Policy | Behavior |
|---|---|
| `no` | Never restart (default) |
| `on-failure[:max-retries]` | Restart only if the container exits with non-zero status |
| `always` | Always restart, even after daemon restart or manual stop... (until explicitly stopped) |
| `unless-stopped` | Like `always`, but won't restart after an explicit manual `docker stop` |

### Resource Constraints
- `--memory` — hard memory limit; exceeding it triggers an **OOM kill** (container gets SIGKILL, exit code 137).
- `--cpus` — limits CPU usage (e.g., `--cpus="1.5"`).
- `--cpu-shares` — relative weighting when CPU is contended (not a hard cap).

Understanding **exit code 137 = OOM kill** is a very common troubleshooting interview question.

### PID 1 Problem
The first process in a container becomes PID 1. Normally the OS's `init` handles reaping zombie processes and forwarding signals — inside a container, if your app isn't designed to do this, you get zombie processes and signals like SIGTERM don't get handled properly (container won't stop gracefully).

**Fix:** run with `docker run --init` (uses `tini` under the hood) or explicitly add `tini`/similar init process as PID 1.

### Exec vs Attach
- `docker exec -it <container> bash` — runs a **new** process inside the running container's namespace (most common way to "get into" a container for debugging).
- `docker attach` — attaches your terminal to the container's **existing** PID 1 process's stdin/stdout (risk: if PID 1 is not shell-friendly, you might accidentally kill it by detaching wrong, or with `Ctrl+C`).

### Logging Drivers
Default is `json-file` (logs stored as JSON on the host, retrievable via `docker logs`). Others: `syslog`, `journald`, `awslogs` (sends logs directly to CloudWatch — directly relevant to your AWS track), `fluentd`, `none`.

---

## 4. Docker Networking

### Definition: Docker Network
A **Docker network** is a virtual networking layer that Docker creates to let containers communicate with each other and with the outside world, while controlling isolation. Every container connects to at least one network, and Docker manages IP allocation, DNS resolution (on user-defined networks), and routing between containers via configurable network drivers (bridge, host, overlay, etc.).

### Network Drivers
| Driver | Use case |
|---|---|
| `bridge` (default) | Default isolated network on a single host; containers get their own IP, NAT'd to host |
| `host` | Container shares the host's network namespace directly — no isolation, no port mapping needed, but no port conflict protection either |
| `none` | No networking at all |
| `overlay` | Multi-host networking (Swarm/orchestrated environments) |
| `macvlan` | Assigns a container a MAC address, making it appear as a physical device on the network |

### Default Bridge vs User-Defined Bridge — the classic gotcha
- **Default bridge network** (`docker0`): containers can reach each other **only by IP**, not by container name. No automatic DNS resolution.
- **User-defined bridge network** (`docker network create mynet`): Docker runs an embedded DNS server, so containers **can resolve each other by container name**. This is why Compose-created networks "just work" with service names.

**Interview answer:** "Always use user-defined networks in practice — automatic service discovery via DNS, better isolation, and you can attach/detach containers without restarting them."

### Port Mapping
`-p hostPort:containerPort` publishes a specific mapping. `-P` publishes all `EXPOSE`d ports to random host ports.

Under the hood: Docker configures `iptables` NAT rules (DNAT) that forward traffic from the host port to the container's internal IP:port on the bridge network.

### Troubleshooting Connectivity
- `docker network inspect <network>` — see which containers are attached and their IPs.
- `docker exec <container> ping <other-container-name>` — test DNS + reachability (only works if on the same user-defined network).
- Common root causes: containers on different networks, using default bridge (no DNS), firewall/iptables rules, or app binding to `127.0.0.1` instead of `0.0.0.0` inside the container.

---

## 5. Storage & Data Persistence

### Definition: Docker Volume
A **Docker volume** is a persistent storage mechanism managed entirely by Docker (stored under `/var/lib/docker/volumes` on the host) that exists independently of any single container's lifecycle. Because containers are ephemeral, volumes are how you keep data (database files, uploads, logs) alive across container restarts, removals, and even recreation from an updated image.

### Volumes vs Bind Mounts vs tmpfs
| Type | Managed by | Use case |
|---|---|---|
| **Volume** | Docker (lives under `/var/lib/docker/volumes`) | Preferred for persistent data (databases, app state); portable, backed up/migrated easily, works well with volume drivers |
| **Bind mount** | You (maps a specific host path) | Local dev — mount source code into a container for live reload; tightly coupled to host filesystem layout |
| **tmpfs** | In-memory only, never written to disk | Sensitive/temporary data (e.g., short-lived secrets) that shouldn't persist |

**Interview framing:** "Volumes for production persistent data since Docker manages lifecycle and portability; bind mounts for local development convenience; tmpfs for sensitive ephemeral data."

### Persistence Patterns
- Stateless containers (most app/API containers) — no persistent volume needed, can be killed/recreated freely.
- Stateful containers (databases, message queues) — must use volumes, and you need a backup/restore strategy since the container itself is disposable but the data isn't.

### Backup/Restore
Common pattern: spin up a temporary container that mounts the same volume, then `tar` the contents out to the host (or to S3).

---

## 6. Docker Compose

### Definition: Docker Compose
**Docker Compose** is a tool for defining and running multi-container Docker applications using a single declarative YAML file (`docker-compose.yml`). Instead of manually running multiple `docker run` commands with matching networks and volumes, Compose lets you describe all your services, their configuration, networks, and volumes in one file and bring the entire stack up or down with a single command (`docker compose up` / `docker compose down`).

### File Structure
```yaml
services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
    env_file:
      - .env
    depends_on:
      - db
    networks:
      - appnet
  db:
    image: postgres:16
    volumes:
      - dbdata:/var/lib/postgresql/data
    networks:
      - appnet
volumes:
  dbdata:
networks:
  appnet:
```

### Environment Variables
- `environment:` — set inline, values visible in `docker-compose.yml`.
- `env_file:` — load from a `.env`-style file, keeps secrets out of version control (should still be gitignored — ties to your Gitleaks workflow).

### depends_on Limitation
`depends_on` only waits for the **container to start**, not for the **application inside to be ready** (e.g., Postgres container started ≠ Postgres accepting connections yet). For a real readiness wait, use `depends_on: condition: service_healthy` combined with a `HEALTHCHECK`, or an app-level retry/wait-for-it script.

### Scaling
`docker compose up --scale web=3` — runs multiple replicas of a service (mainly useful for local testing of load-balancing behavior; real scaling belongs to an orchestrator).

### Compose vs Production Reality
Compose is excellent for local dev and small single-host setups but lacks: multi-host scheduling, self-healing, rolling updates, and horizontal autoscaling. This is the natural bridge to "why Kubernetes" — good to have this articulated clearly.

---

## 7. Security

Directly relevant given your Trivy/Gitleaks hands-on experience — expect interviewers to dig into your actual workflow here.

### Run as Non-Root
By default containers run as root inside the container, which — combined with certain misconfigurations — can lead to privilege escalation onto the host. Best practice:
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### Image Vulnerability Scanning (Trivy)
Trivy scans image layers against CVE databases for OS packages and language dependencies. Typical pipeline placement: **after build, before push** — fail the pipeline on HIGH/CRITICAL findings above a threshold. Be ready to describe: what severity threshold you used, whether you scanned base images too, and how you handled false positives/exceptions.

### Secrets Management
Never do this:
```dockerfile
ENV API_KEY=supersecret   # BAD — baked into image layer, visible via docker history
```
Better patterns:
- BuildKit secret mounts (`--mount=type=secret`) — available only during that RUN step, never persisted in a layer.
- External secrets manager (AWS Secrets Manager / SSM Parameter Store) — inject at container runtime, not build time.
- Gitleaks in your pipeline — scans source/Dockerfiles for accidentally committed secrets before they ever get near a build.

### Read-Only Root Filesystem
`docker run --read-only` — prevents writes to the container filesystem at runtime, reducing the blast radius if an attacker gains code execution. Pair with explicit writable volumes/tmpfs only where truly needed (e.g., `/tmp`).

### Docker Content Trust / Image Signing
Cryptographically signs images so consumers can verify authenticity/integrity before pulling — conceptual awareness is usually enough at entry level.

### Capabilities and --privileged
Linux capabilities let you grant a container specific elevated permissions (e.g., `NET_ADMIN`) without giving full root. `--privileged` disables almost all isolation and gives the container near-host-level access — avoid in production; only used for things like Docker-in-Docker or specific hardware access.

### Minimizing Attack Surface
Combine: multi-stage builds (drop build tools) + minimal/distroless base images + non-root user + read-only filesystem + regular Trivy scans in CI.

---

## 8. Registries

- **Docker Hub** — public default registry; rate limits apply for anonymous/free pulls (relevant for CI pipelines that pull frequently).
- **Private registries** — Amazon ECR, GCR, Harbor, self-hosted. ECR is the one most relevant to your AWS track — integrates with IAM for auth, supports image scanning and lifecycle policies natively.
- **Auth workflow:** `docker login`, or for ECR: `aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com`.
- **Tagging conventions in CI/CD:** avoid relying on `latest` in production — tag with commit SHA, semantic version, or both, so deployments are traceable and rollback-able.
- **Retention policies:** ECR lifecycle policies to auto-expire old/untagged images and control storage cost.

---

## 9. Docker in CI/CD

Directly relevant to your GitHub Actions/Jenkins experience.

- **Build caching in pipelines:** use `docker buildx` with registry cache (`--cache-from`/`--cache-to`) or GitHub Actions cache backend, so repeated pipeline runs don't rebuild unchanged layers from scratch.
- **Tagging strategy:** tag images with `${GITHUB_SHA}` or `${{ github.ref_name }}` for traceability; optionally also tag `latest` for convenience on main branch only.
- **Push to ECR from GitHub Actions:** typical flow is `aws-actions/configure-aws-credentials` → `aws ecr get-login-password` → `docker build` → `docker push`.
- **Scanning stage:** run Trivy against the built image as a pipeline step *before* the push step, gating the pipeline on severity threshold.
- **Multi-arch builds:** `docker buildx build --platform linux/amd64,linux/arm64` — relevant if deploying to mixed architecture (e.g., Graviton instances on AWS).

---

## 10. Docker Swarm

### Definition: Docker Swarm
**Docker Swarm** is Docker's native container orchestration tool, built directly into the Docker Engine, that turns a group of Docker hosts into a single logical cluster. It handles scheduling containers across multiple nodes, scaling services, load balancing, and self-healing (restarting failed tasks) — essentially Docker's own answer to "what Kubernetes does," but with a much simpler feature set and gentler learning curve.

- **Service** — declarative spec of a task to run (like a Deployment in K8s, but simpler).
- **Task** — single container instance of a service.
- **Stack** — group of services deployed together (Swarm's version of Compose in production, using the same Compose file syntax).

**Why the industry moved to Kubernetes:** Swarm is simpler but far less feature-rich — no advanced scheduling, limited autoscaling, smaller ecosystem, fewer extensibility points (no CRDs/operators), and weaker multi-cloud tooling support. Kubernetes won the orchestration space due to ecosystem maturity (Helm, Operators, CNCF tooling) and cloud-provider-native support (EKS, GKE, AKS).

---

## 11. Troubleshooting & Real-World Scenarios

- **Container exits immediately:** check `docker logs <container>` first, then `docker inspect <container>` for the exit code. Exit code 0 = clean exit (often means the main process finished and there's nothing keeping it alive — common with missing `CMD`/wrong entrypoint). Exit code 1 = general application error. Exit code 137 = OOM killed. Exit code 143 = SIGTERM (often from `docker stop`).
- **Container can't reach another container:** verify both are on the same **user-defined** network (`docker network inspect`), verify DNS resolution by name works, check the app is listening on `0.0.0.0` not `127.0.0.1` inside the container.
- **Image too large:** use `docker history` to find the fat layer, switch to multi-stage build, use a slimmer/distroless base, add a proper `.dockerignore`, combine `RUN` instructions to reduce layer count, clean up package manager cache in the same `RUN` step it was created in.
- **"Works on my machine":** compare image digests, check for reliance on host-installed dependencies that aren't in the image, check environment variable differences, verify base image tags aren't floating (`latest` drifting between builds).
- **High memory usage / OOM kills:** check `docker stats` for live usage, review `--memory` limits vs actual app requirements, check for memory leaks in the app itself, review if the app is aware of the cgroup limit (older JVMs, for example, historically ignored cgroup limits and used host memory as their basis).
- **Disk space filling up:** `docker system df` to see breakdown, `docker system prune` (careful — removes stopped containers, unused networks, dangling images), `docker image prune -a` for all unused images, `docker volume prune` for orphaned volumes.

---

## 12. Docker → Kubernetes Bridge

- **Why containers alone aren't production-ready:** no self-healing (a crashed container just stays dead unless a restart policy is set — and that's single-host only), no multi-host scheduling, no built-in service discovery/load balancing across hosts, no rolling updates/rollback mechanism, no horizontal autoscaling.
- **Image → Pod:** a Kubernetes Pod wraps one or more containers (usually one) with shared network/storage namespace, and Kubernetes manages the Pod's lifecycle (scheduling, restarts, scaling) rather than you manually running `docker run`.
- **Networking model differences:** Docker's default bridge network is single-host and NAT-based; Kubernetes assumes a **flat network** where every Pod gets its own routable IP and Pods can reach each other directly without NAT (implemented differently per CNI plugin — Calico, AWS VPC CNI in EKS's case).
- **containerd and dockershim removal:** Kubernetes originally used Docker via a shim (`dockershim`) to satisfy the Container Runtime Interface (CRI). Since Docker's own daemon doesn't natively speak CRI, and containerd (which IS CRI-compliant and IS what Docker uses under the hood) already sat underneath Docker, Kubernetes dropped the shim (v1.24+) and talks to containerd directly. **Your images are unaffected** — this is a control-plane implementation detail, not a change to image format (both still OCI-compliant).

---

## Scenario-Based Questions

**Q1: Your container keeps restarting in a crash loop. Walk me through your debugging steps.**
Check `docker ps -a` for exit code and restart count → `docker logs` for the actual error → `docker inspect` for restart policy and resource limits → if exit code 137, check memory limits vs actual usage → if it's a config/env issue, verify env vars are actually being passed correctly (`docker inspect` also shows resolved env vars).

**Q2: A teammate says "just add `latest` tag and it'll always be fine." Why is this risky in production?**
`latest` is not guaranteed to be the newest version — it's just a tag that floats to whatever was last pushed without an explicit tag. It breaks reproducibility (you can't know exactly what's running), makes rollbacks harder (no clear previous version to revert to), and can cause cache confusion in CI/CD. Use immutable tags like commit SHA or semantic version instead.

**Q3: Your image build is taking 8 minutes and most of it is `npm install` / `pip install` running every time, even for small code changes. How do you fix it?**
Reorder the Dockerfile so dependency manifest files (`package.json`, `requirements.txt`) are copied and installed **before** copying the full source code — that way the dependency-install layer stays cached unless the manifest itself changes. Also consider BuildKit cache mounts for the package manager's own cache directory.

**Q4: Two containers on the default bridge network can't resolve each other by name — why, and how do you fix it?**
The default bridge network doesn't provide automatic DNS-based service discovery — only IP-based reachability. Fix: create and use a user-defined bridge network (or use Docker Compose, which creates one automatically), which runs an embedded DNS server for name resolution.

**Q5: How would you reduce a 1.2GB Node.js image down significantly?**
Multi-stage build (build stage with full toolchain, runtime stage copying only `node_modules` and built assets), switch to an `alpine` or `slim` base for the runtime stage, ensure `.dockerignore` excludes `node_modules`/`.git` from build context, and run `npm ci --production` (or equivalent) to skip devDependencies in the final layer.

**Q6: How do you handle secrets when building an image that needs to `npm install` from a private registry requiring an auth token?**
Use a BuildKit secret mount (`RUN --mount=type=secret,id=npmtoken npm install`) so the token is available only during that build step and never persisted into any layer or the final image history — as opposed to passing it via `ARG`/`ENV`, which would leak it into `docker history`.

**Q7: In your pipeline, where exactly does Trivy scanning sit, and what happens on a critical finding?**
After the image is built, before it's pushed to the registry — scan the freshly built image, and if findings meet or exceed your defined severity threshold (typically HIGH/CRITICAL), fail the pipeline so the vulnerable image never reaches the registry or a deployment.

---

## Core Interview Q&A

**Q: What's the difference between a container and a VM?**
A: VMs virtualize hardware and run a full guest OS on a hypervisor; containers virtualize the OS and share the host kernel, isolated via namespaces and cgroups — making them far lighter and faster to start.

**Q: Explain CMD vs ENTRYPOINT.**
A: CMD provides default arguments/command that can be fully overridden at `docker run`. ENTRYPOINT is the fixed executable; anything passed at runtime is appended as arguments to it rather than replacing it.

**Q: What happens when a container exceeds its memory limit?**
A: The kernel's OOM killer terminates the process, and the container exits with code 137.

**Q: Why use multi-stage builds?**
A: To keep build-time dependencies (compilers, dev tools) out of the final runtime image, drastically reducing image size and attack surface while keeping the Dockerfile in a single, maintainable file.

**Q: What's the difference between a volume and a bind mount?**
A: A volume is fully managed by Docker (storage location, backup, portability) and is the recommended approach for persistent data. A bind mount maps a specific host path directly into the container, useful for local development but tightly coupled to the host's filesystem.

**Q: Why did Kubernetes drop dockershim?**
A: Kubernetes communicates with container runtimes via the CRI (Container Runtime Interface). Docker's daemon doesn't natively implement CRI, so Kubernetes used a shim to translate. Since containerd (which Docker itself uses underneath) is natively CRI-compliant, Kubernetes removed the shim and talks to containerd directly — images and Dockerfiles are unaffected since both remain OCI-compliant.

**Q: How do you keep secrets out of a Docker image?**
A: Never use `ENV`/`ARG` for secrets since they persist in layer history and `docker inspect`. Use BuildKit secret mounts for build-time secrets, and inject runtime secrets via an external secrets manager (e.g., AWS Secrets Manager/SSM) rather than baking them into the image.

**Q: What's the difference between `docker stop` and `docker kill`?**
A: `stop` sends SIGTERM and waits a grace period (default 10s) for graceful shutdown before sending SIGKILL. `kill` sends SIGKILL (or a specified signal) immediately, with no grace period.
