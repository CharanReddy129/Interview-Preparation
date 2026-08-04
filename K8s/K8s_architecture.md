# Kubernetes Notes — Part 1
## Core Concepts & Architecture

---

### What is Kubernetes?
**Kubernetes (K8s)** is an open-source **container orchestration platform** that automates the deployment, scaling, networking, and lifecycle management of containerized applications across a cluster of machines. Where Docker packages and runs a *single container on a single host*, Kubernetes takes that same container image and manages it across a *fleet of hosts* — deciding where it runs, keeping it running, scaling it up or down, routing traffic to it, and replacing it automatically if it fails.

In one line: *Kubernetes is the layer above Docker that turns "run this container" into "keep this application running, healthy, and scaled, across many machines, without me manually intervening."*

The name comes from the Greek word for "helmsman" (the person who steers a ship) — origin of the "K8s" abbreviation (K + 8 letters + s).

### Why Kubernetes — Problems It Solves That Docker/Compose/Swarm Alone Can't
- **Multi-host scheduling** — Docker alone runs containers on a single host. Compose is single-host too. Kubernetes schedules containers (as Pods) across an entire cluster of nodes, picking the best-fit node based on resources.
- **Self-healing at scale** — a container crashing on one host with plain Docker just stays dead until someone (or a restart policy) intervenes. Kubernetes continuously reconciles actual state vs desired state — if a Pod dies, it's automatically rescheduled, cluster-wide.
- **Horizontal autoscaling** — Kubernetes can automatically add/remove Pod replicas based on real-time CPU/memory/custom metrics (Horizontal Pod Autoscaler), and even add/remove entire nodes (Cluster Autoscaler/Karpenter). Docker/Compose have no native equivalent.
- **Service discovery & load balancing across hosts** — Docker's default networking is single-host; Kubernetes provides a flat, cluster-wide network where any Pod can reach any Service by name, with built-in load balancing across replicas.
- **Rolling updates & rollbacks** — Kubernetes Deployments update Pods gradually with zero downtime and can roll back to a previous revision with a single command; Docker Swarm has a basic version of this, Compose/plain Docker essentially don't.
- **Ecosystem and extensibility** — Kubernetes has a massive ecosystem (Helm, Operators, CRDs, service meshes, GitOps tooling) and native support from every major cloud provider (EKS, GKE, AKS) — Swarm never reached this level of adoption or tooling maturity.

**Interview framing:** "Docker solves 'how do I package and run one container consistently.' Kubernetes solves 'how do I run hundreds of containers, across many machines, reliably, at scale, without babysitting them.'"

### Real-World Uses
- Running production microservices architectures where dozens/hundreds of services need independent scaling and deployment.
- Blue-green and canary deployments for safe, gradual rollouts of new versions.
- Auto-scaling workloads that see variable traffic (e-commerce during sales, batch processing jobs).
- Multi-tenant platforms where different teams/environments share a cluster but stay isolated via namespaces and RBAC.
- Running stateful workloads (databases, message queues) with stable identity and persistent storage via StatefulSets.
- Serving as the deployment target for GitOps pipelines (Argo CD) and CI/CD workflows (GitHub Actions → build → push to ECR → deploy to EKS).
- Hybrid/multi-cloud deployments, since Kubernetes provides a consistent API regardless of underlying infrastructure.

### Key Features of Kubernetes
- **Self-healing** — automatically restarts failed containers, replaces and reschedules Pods when nodes die, kills and restarts containers that fail health checks.
- **Horizontal scaling** — scale applications up/down manually, automatically via CPU/memory usage, or via custom metrics.
- **Declarative configuration** — you describe the *desired state* (via YAML manifests) and Kubernetes continuously works to make reality match it, rather than you scripting each step imperatively.
- **Service discovery & load balancing** — built-in DNS-based service discovery and automatic load balancing across healthy Pod replicas.
- **Automated rollouts & rollbacks** — change a Deployment's spec and Kubernetes gradually rolls out the change while monitoring health, with an easy rollback path.
- **Storage orchestration** — automatically mounts local storage, cloud storage (EBS/EFS), or network storage as needed.
- **Portability** — the same manifests can run on any conformant Kubernetes distribution — on-prem, EKS, GKE, AKS, minikube — avoiding cloud lock-in at the orchestration layer.
- **Extensibility** — Custom Resource Definitions (CRDs) and Operators let you extend Kubernetes' API to manage virtually anything declaratively.
- **Secret & configuration management** — built-in objects (ConfigMaps, Secrets) to manage app configuration separately from container images.
- **Batch execution support** — native Job/CronJob objects for one-off and scheduled batch workloads.

---

## Core Concepts & Architecture

### Kubernetes Cluster Architecture: Control Plane vs Worker Nodes
![K8s Architecture](https://media.geeksforgeeks.org/wp-content/uploads/20260406154006746759/k8s-arch.webp)

A Kubernetes **cluster** is made up of two categories of machines:
- **Control Plane** — the "brain" of the cluster. Makes global decisions (scheduling, detecting/responding to cluster events), maintains the desired state, and exposes the API that `kubectl` and everything else talks to.
- **Worker Nodes** — the machines that actually run your application Pods. Each node runs the components needed to receive instructions from the control plane and run containers accordingly.

A production cluster typically runs multiple control plane instances (for high availability) and multiple worker nodes (for capacity and fault tolerance).

### Control Plane Components
- **API Server (`kube-apiserver`)** — the front door to the cluster. Every interaction — `kubectl`, controllers, kubelets, CI/CD pipelines — goes through the API server via REST calls. It validates requests and persists state to etcd.
- **etcd** — a distributed, consistent key-value store that holds the entire cluster's state (all objects, their specs and statuses). It is the single source of truth — if etcd is lost without backup, the cluster's state is effectively lost.
- **Scheduler (`kube-scheduler`)** — watches for newly created Pods with no assigned node, and decides which node they should run on based on resource requirements, constraints (affinity/taints/tolerations), and current cluster load.
- **Controller Manager (`kube-controller-manager`)** — runs the various controllers that continuously reconcile actual state to desired state (e.g., the Deployment controller ensures the right number of Pod replicas exist, the Node controller notices when a node goes down).
- **Cloud Controller Manager** — integrates the cluster with the underlying cloud provider's APIs (e.g., provisioning an AWS ELB when you create a `LoadBalancer` Service, managing cloud-specific node lifecycle). This is what lets EKS hook into real AWS infrastructure.

### Node (Worker) Components
- **kubelet** — the agent running on every node; talks to the API server, receives Pod specs, and ensures the containers described in those specs are actually running and healthy on that node (via the container runtime).
- **kube-proxy** — maintains network rules on each node that implement the Kubernetes Service abstraction — routing traffic destined for a Service's virtual IP to the correct backend Pod(s), typically via iptables or IPVS rules.
- **Container runtime (containerd)** — the actual engine that pulls images and runs containers on the node, invoked by the kubelet via the Container Runtime Interface (CRI). Modern Kubernetes uses containerd directly (post-dockershim removal) — ties back to the Docker→K8s bridge topic.

### Declarative vs Imperative Management
- **Imperative** — you tell Kubernetes exactly what action to take, step by step (e.g., `kubectl run nginx --image=nginx`, `kubectl scale deployment nginx --replicas=5`). Fast for one-off tasks, but not easily version-controlled or repeatable.
- **Declarative** — you describe the *desired end state* in a YAML manifest (e.g., "there should always be 5 replicas of this Pod") and apply it (`kubectl apply -f deployment.yaml`). Kubernetes' controllers continuously run a **reconciliation loop**: compare desired state (from etcd) to actual observed state, and take action to close any gap. This is the standard approach in real-world/production and GitOps workflows — it's what makes tools like Argo CD possible.

**Interview framing:** "Declarative is preferred in production because it's idempotent, version-controllable in Git, and self-correcting — if someone manually changes something, the reconciliation loop notices and fixes the drift."

### kubectl and the API Server
`kubectl` is the CLI tool used to interact with a Kubernetes cluster. Every `kubectl` command is translated into a REST API call to the **API server** — `kubectl` itself has no direct access to etcd, nodes, or containers; it's purely a client of the API server. Authentication/authorization happens at this API server layer (via kubeconfig, RBAC, etc.), which is why access control questions in interviews almost always trace back to "the API server is the single gatekeeper."

### YAML Manifests — Structure
Every Kubernetes object is described declaratively with four top-level fields:
```yaml
apiVersion: apps/v1        # which version of the Kubernetes API this object uses
kind: Deployment            # the type of object (Pod, Service, Deployment, etc.)
metadata:                   # identifying info — name, namespace, labels, annotations
  name: my-app
  labels:
    app: my-app
spec:                       # the desired state — what you want this object to look like/do
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:                 # for workload objects, this is the Pod template
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0
```
- `apiVersion` — differs by object maturity/group (e.g., `v1` for core objects like Pod/Service, `apps/v1` for Deployment/StatefulSet/DaemonSet, `batch/v1` for Job/CronJob).
- `kind` — the object type being defined.
- `metadata` — name, namespace, labels (used for selection/grouping), annotations (non-identifying metadata).
- `spec` — the desired state; structure varies significantly by `kind` (this is what you spend most of your time editing).
- (Read-only) `status` — populated by Kubernetes itself to reflect the *actual* observed state; you never write this, only read it (e.g., via `kubectl get`/`describe`).

---

## Scenario-Based Questions

**Q1: A teammate says "why not just use Docker Swarm since it's simpler and it's already built into Docker?" How do you respond?**
Swarm is simpler and has a gentler learning curve, but Kubernetes wins on ecosystem maturity, extensibility (CRDs/Operators), advanced scheduling (affinity/taints/tolerations), robust autoscaling, and near-universal cloud-provider support (EKS/GKE/AKS). For anything beyond small/simple deployments, or anywhere long-term hiring and tooling support matters, Kubernetes is the safer choice — which is also why the industry consolidated around it.

**Q2: You run `kubectl apply -f deployment.yaml` and then manually edit a Pod's environment variable directly with `kubectl edit`. What happens over time?**
Kubernetes' reconciliation loop will eventually notice the actual state (manually edited Pod) doesn't match the desired state stored in etcd (from the original manifest) and can revert or recreate the Pod to match the declared spec, depending on the object type — manual imperative changes to fields managed by a controller tend to get overwritten. This is exactly why GitOps workflows insist all changes go through version-controlled manifests, not manual edits.

**Q3: `kubectl get pods` is hanging or returning a connection error across your whole team. Where do you start investigating?**
Since `kubectl` only talks to the API server, this points to the API server (or the network path to it) rather than individual nodes/Pods. Check API server health/availability first (in EKS, check the control plane status in the AWS console/cluster health), then check your kubeconfig/network connectivity, before assuming anything is wrong with workloads themselves.

**Q4: Your cluster's etcd data gets corrupted with no backup. What's the practical impact?**
etcd is the single source of truth for the entire cluster's state — losing it without a backup means losing all object definitions (Deployments, Services, ConfigMaps, everything) the control plane knows about, even if the actual containers happen to still be running on nodes momentarily. This is why etcd backup strategy is a critical operational concern (mitigated for you on EKS since AWS manages control plane/etcd, but important to understand conceptually).

---

## Core Interview Q&A

**Q: What is Kubernetes, in one sentence?**
A: An open-source container orchestration platform that automates deploying, scaling, healing, and networking containerized applications across a cluster of machines.

**Q: What's the difference between the Control Plane and Worker Nodes?**
A: The Control Plane makes cluster-wide decisions and stores desired state (API Server, etcd, Scheduler, Controller Manager); Worker Nodes actually run the application containers, managed by the kubelet and kube-proxy on each node.

**Q: What is etcd and why does it matter?**
A: etcd is a distributed key-value store that holds the entire cluster's state — every object's spec and status. It's the single source of truth; if it's lost without backup, the cluster loses all record of what should be running.

**Q: What does the kube-scheduler actually do?**
A: It watches for Pods that have no node assigned yet and decides which node to place them on, based on resource availability, constraints like affinity/anti-affinity, and taints/tolerations.

**Q: Explain declarative vs imperative management in Kubernetes.**
A: Imperative means issuing direct commands for specific actions (`kubectl scale --replicas=5`). Declarative means describing the desired end state in a manifest and letting Kubernetes' controllers continuously reconcile actual state to match it — the standard, GitOps-friendly approach in production.

**Q: Does kubectl talk directly to nodes or containers?**
A: No — kubectl only communicates with the API server via REST calls. The API server is the single entry point for all cluster interactions; it's responsible for authentication, authorization, and persisting changes to etcd.

**Q: What replaced dockershim, and why?**
A: containerd is used directly. Kubernetes talks to container runtimes via the Container Runtime Interface (CRI); Docker's daemon didn't natively implement CRI, so a shim was used to translate. Since containerd (which sits underneath Docker anyway) is natively CRI-compliant, the shim was removed in Kubernetes v1.24+ — this doesn't affect your images or Dockerfiles since both remain OCI-compliant.
