# Kubernetes Notes — Part 2
---

## Pods & Workload Objects

### Definition: Pod
A **Pod** is the smallest deployable unit in Kubernetes — a wrapper around one or more tightly coupled containers that share the same network namespace (same IP, same port space, can talk via `localhost`) and can share storage volumes. You never deploy a "raw container" in Kubernetes — everything runs inside a Pod, even if it's just one container.

**Single-container Pods** are the norm — one container per Pod, one concern per Pod, matching microservice design.

**Multi-container Pods** exist for containers that must be co-located and co-scheduled — they always start, stop, and scale together, on the same node. The most common pattern:
- **Sidecar pattern** — a helper container running alongside the main application container, extending or supporting it without changing its code. Common real-world examples: a log-shipping sidecar (reads logs from a shared volume and forwards them to a logging backend), a service mesh proxy (e.g., Envoy in Istio, injected as a sidecar into every Pod), or a config-reloader sidecar that watches for ConfigMap changes.

**Interview framing:** "Multi-container Pods are for containers that are tightly coupled and must scale as a unit — not just 'related' services. If two containers can reasonably be scaled or deployed independently, they belong in separate Pods."

**Example — single-container Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

**Example — multi-container Pod with a sidecar:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logging-sidecar
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: log-shipper
      image: fluent-bit:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
  volumes:
    - name: shared-logs
      emptyDir: {}
```

### ReplicaSet
A **ReplicaSet** ensures a specified number of identical Pod replicas are running at any given time. If a Pod dies, the ReplicaSet notices (via its controller reconciliation loop) and creates a replacement; if there are too many, it terminates the excess. In practice, you almost never create a ReplicaSet directly — you create a **Deployment**, which manages ReplicaSets for you underneath.

**Example:**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

### Deployment

**Deployment** is a higher-level Kubernetes object that wraps around a Pod and adds features such as self-healing, scaling, zero-downtime rollouts, and versioned rollbacks.

A **Deployment** is the standard way to manage stateless applications in Kubernetes. It manages ReplicaSets on your behalf and adds:
- **Rolling updates** — when you change the Pod template (e.g., new image version), the Deployment gradually replaces old Pods with new ones (spinning up new Pods, waiting for them to become ready, then terminating old ones), avoiding downtime. Controlled via `strategy.rollingUpdate.maxSurge` / `maxUnavailable`.
- **Rollback** — if a new rollout is broken, `kubectl rollout undo deployment/<name>` reverts to the previous working revision.
- **Revision history** — Kubernetes keeps a history of ReplicaSets from past rollouts (`kubectl rollout history deployment/<name>`), letting you roll back to a specific prior revision, not just the immediately previous one.

Under the hood: `Deployment → manages → ReplicaSet(s) → manages → Pods`. Each rollout creates a new ReplicaSet while scaling the old one down to zero.

**Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.2.0
          ports:
            - containerPort: 8080
```

### StatefulSet
A **StatefulSet** manages Pods that need **stable, unique identity** and **stable storage**, unlike Deployments where Pods are interchangeable/disposable. Key differences from Deployment:
- Pods get **stable, predictable names** (`web-0`, `web-1`, `web-2` — not random suffixes).
- Pods are created/scaled/deleted **in order** (0, then 1, then 2...) and terminated in reverse order.
- Each Pod gets its **own PersistentVolumeClaim** that persists across rescheduling — Pod `web-1` always reattaches to the same volume it had before, even if rescheduled to a different node.
- Requires a **headless Service** to provide stable network identities (`web-0.web-service.namespace.svc.cluster.local`).

**Use case:** databases, message queues (Kafka, Zookeeper), or any clustered system where node identity and data affinity matter — a replica needs to know "I am replica 1" consistently.

**Example:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web-service"   # must point to a headless Service
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: my-db:1.0
          volumeMounts:
            - name: data
              mountPath: /var/lib/data
  volumeClaimTemplates:        # each Pod gets its own PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
```

### DaemonSet
A **DaemonSet** ensures that a copy of a specific Pod runs on **every node** (or a selected subset of nodes) in the cluster — as nodes are added, the DaemonSet automatically schedules a Pod onto them; as nodes are removed, those Pods are cleaned up.

**Use case:** node-level infrastructure agents — log collectors (Fluentd/Fluent Bit), monitoring agents (Prometheus Node Exporter, Datadog agent), or network/storage plugins that must run on every node.

**Example:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:latest
```

### Job and CronJob
- **Job** — runs one or more Pods to completion for a finite, one-off task (not a long-running service). Kubernetes tracks successful completions and can retry failed Pods up to a configured limit.
- **CronJob** — creates Jobs on a repeating schedule, using standard cron syntax (e.g., `0 2 * * *` for daily at 2 AM). Used for scheduled batch tasks — nightly backups, report generation, cleanup scripts.

**Example — Job:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
        - name: migrate
          image: my-app-migrator:1.0
          command: ["./migrate.sh"]
      restartPolicy: Never
```

**Example — CronJob:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"     # every day at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: my-backup-tool:1.0
          restartPolicy: OnFailure
```

### Init Containers
**Init containers** run **before** the main application container(s) in a Pod starts, and must run to completion successfully before the main containers are started. Unlike sidecars, they don't run alongside the app — they run first, sequentially, then exit.

**Use cases:** waiting for a dependency to become available (e.g., waiting for a database to be reachable), running a one-time setup/migration step, pulling configuration/secrets into a shared volume before the main container needs them.

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ["sh", "-c", "until nc -z db-service 5432; do sleep 2; done"]
  containers:
    - name: app
      image: my-app:1.0
```

### Pod Lifecycle Phases and Container States
**Pod phases** (`status.phase` — high-level, coarse):
| Phase | Meaning |
|---|---|
| `Pending` | Pod accepted by the cluster, but one or more containers not yet running (e.g., still scheduling, or pulling images) |
| `Running` | Pod bound to a node, all containers created, at least one running |
| `Succeeded` | All containers terminated successfully, won't restart (typical for Jobs) |
| `Failed` | All containers terminated, at least one failed |
| `Unknown` | Pod state can't be determined (usually a communication issue with the node) |

**Container states** (per-container, more granular — visible via `kubectl describe pod`):
| State | Meaning |
|---|---|
| `Waiting` | Container not yet running — still pulling its image, waiting on init containers, or blocked by an error (e.g., `ImagePullBackOff`, `CrashLoopBackOff` show up here) |
| `Running` | Container executing without issues |
| `Terminated` | Container finished execution (successfully or with an error) — includes exit code and reason |

**Interview framing:** "Pod phase gives you the big picture; container state (via `kubectl describe`) gives you the actual reason something's wrong — that's always my second debugging step after `kubectl get pods`."

---

## Networking

### Definition: Kubernetes Networking Model
Kubernetes networking is built on a **flat network model**: every Pod in the cluster gets its own unique, routable IP address, and **every Pod can reach every other Pod directly, without NAT**, regardless of which node they're on. This is fundamentally different from Docker's default single-host, NAT-based bridge networking.

The three core rules of the Kubernetes networking model:
1. Every Pod gets its own IP.
2. Pods on any node can communicate with all Pods on all nodes without NAT.
3. Agents on a node (like the kubelet) can communicate with all Pods on that node.

This flat model is implemented by a **CNI (Container Network Interface) plugin** — Kubernetes itself doesn't implement the networking, it delegates to a plugin that satisfies these rules (Calico, AWS VPC CNI, etc.).

### Definition: Service
A **Service** is a stable networking abstraction that provides a single, consistent endpoint (virtual IP + DNS name) for a group of Pods, selected via label selectors. Since Pods are ephemeral and their IPs change every time they're recreated, Services solve "how do I reliably reach a set of Pods that keep changing identity."

**Service types:**
| Type | Behavior |
|---|---|
| `ClusterIP` (default) | Exposes the Service on an internal-only virtual IP, reachable only from inside the cluster |
| `NodePort` | Exposes the Service on a static port on every node's IP, making it reachable from outside the cluster via `<NodeIP>:<NodePort>` |
| `LoadBalancer` | Provisions an external cloud load balancer (e.g., an AWS ELB/NLB via the Cloud Controller Manager) that routes to the Service — the standard way to expose a Service publicly on a cloud provider |
| `ExternalName` | Maps the Service to an external DNS name (e.g., a managed database outside the cluster) via a CNAME — no proxying, just DNS-level redirection |

**Example — ClusterIP (default):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

**Example — NodePort:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080     # must be in range 30000-32767
```

**Example — LoadBalancer:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

**Example — ExternalName:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: mydb.abc123.us-east-1.rds.amazonaws.com
```

### Ingress and Ingress Controllers
A **Service** (even `LoadBalancer`) typically maps to one application. **Ingress** is a higher-level object that manages **HTTP/HTTPS routing** into the cluster — host-based and path-based routing to multiple backend Services through a single entry point, plus TLS termination.

Ingress itself is just a set of routing rules — it does nothing on its own. You need an **Ingress Controller** (e.g., NGINX Ingress Controller, or the AWS Load Balancer Controller for EKS, which provisions an ALB) actually running in the cluster to read Ingress objects and configure the underlying proxy/load balancer accordingly.

**Interview framing:** "Without a Service, Pods aren't reliably reachable. Without an Ingress + Ingress Controller, you'd need a separate LoadBalancer Service (and separate cloud load balancer) per application — Ingress lets many apps share one entry point and one load balancer, routed by hostname/path."

**Example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### kube-proxy and Service Routing
**kube-proxy** runs on every node and is responsible for actually implementing the Service abstraction at the network level — it watches the API server for Service and Endpoint changes and programs routing rules accordingly so that traffic sent to a Service's virtual IP gets transparently forwarded to one of the backing Pods.

Two main modes:
- **iptables mode** (long-time default) — uses Linux iptables rules to randomly select a backend Pod for each connection. Simple, but rule evaluation is roughly linear with Service count, which can become a bottleneck at very large scale.
- **IPVS mode** — uses the Linux kernel's IPVS (IP Virtual Server), which uses hash tables instead of sequential rule matching, offering better performance at scale and more load-balancing algorithm choices (round robin, least connection, etc.).

### CoreDNS and Service Discovery
**CoreDNS** is the cluster's internal DNS server (runs as Pods within the cluster itself) that provides **service discovery by name**. Every Service automatically gets a DNS record in the form:
```
<service-name>.<namespace>.svc.cluster.local
```
So a Pod in the same namespace can simply reach another Service by its short name (`my-service`); across namespaces, the fully qualified form is needed. This is the mechanism that lets application code use human-readable service names instead of hardcoded IPs — directly mirroring the "user-defined network DNS" concept from Docker, but cluster-wide instead of single-host.

### Network Policies
By default, Kubernetes networking is **fully open** — any Pod can talk to any other Pod, cluster-wide, with no restrictions. A **NetworkPolicy** is how you explicitly restrict pod-to-pod traffic — defining rules (based on label selectors, namespaces, or IP blocks) for which ingress/egress traffic is allowed to/from a given set of Pods.

**Important caveat:** NetworkPolicies only take effect if the CNI plugin supports enforcing them — not all CNI plugins do (the basic AWS VPC CNI historically needed enforcement via Calico or the newer network policy support added to it; always verify plugin capability rather than assuming policies are enforced).

**Use case:** enforcing that only the frontend Pods can talk to the backend Pods, or that database Pods only accept traffic from application Pods and nothing else — a core security control at the pod-to-pod level.

**Example — only allow traffic to `db` Pods from `backend` Pods:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
```

### CNI Plugins
The **Container Network Interface (CNI)** is the standard interface Kubernetes uses to delegate actual network setup (IP allocation, routing, and often network policy enforcement) to a pluggable implementation.

- **Calico** — widely used, supports NetworkPolicy enforcement robustly, works across many environments (on-prem and cloud).
- **AWS VPC CNI** — the default CNI on EKS; it's distinctive because it assigns **real VPC IP addresses directly to Pods** (rather than an overlay network), meaning Pods are natively routable within your AWS VPC and can be reached (and secured) using standard VPC networking constructs like security groups — this is a key EKS-specific detail worth knowing given your background.

---

## Scenario-Based Questions

**Q1: You need to run a logging agent that reads log files written by your application container and ships them elsewhere. How would you architect this in Kubernetes?**
Use a sidecar container within the same Pod as the application, sharing a volume where the app writes logs and the sidecar reads/ships them — since they need to be co-scheduled and share storage, but each has a distinct, independently-updatable responsibility (that reasoning also justifies why it's a sidecar and not just baked into the app container).

**Q2: A stateful clustered database (like a 3-node Kafka setup) needs each replica to always reconnect to its own data after a restart. Which workload object do you use, and why not a Deployment?**
StatefulSet — because each Pod needs a stable, predictable identity (`kafka-0`, `kafka-1`, `kafka-2`) and its own dedicated PersistentVolumeClaim that follows it across rescheduling. A Deployment's Pods are interchangeable and don't guarantee this stable identity/storage binding, which would break a clustered stateful system that depends on knowing which replica is which.

**Q3: Your Pod shows phase `Pending` and has been stuck there for several minutes. What do you check?**
`kubectl describe pod <name>` for events — most commonly this is insufficient cluster resources (no node has enough CPU/memory to satisfy the Pod's requests), unschedulable due to taints/node affinity rules with no matching node, or a PersistentVolumeClaim that hasn't been bound yet.

**Q4: You need to expose three different applications on the same domain but different paths (`/api`, `/app`, `/admin`) using a single load balancer. What do you use, and why not three separate LoadBalancer Services?**
Ingress + an Ingress Controller — three separate `LoadBalancer` Services would provision three separate cloud load balancers (unnecessary cost and complexity). A single Ingress object can route by path to three different backend Services through one Ingress Controller-managed load balancer.

**Q5: Two Pods in different namespaces can't reach each other by short DNS name (`my-service`) but can by the fully qualified name. Is this a bug?**
No — this is expected DNS scoping behavior. The short form `<service-name>` only resolves within the same namespace; cross-namespace calls require `<service-name>.<namespace>.svc.cluster.local` (or at minimum `<service-name>.<namespace>`).

**Q6: You applied a NetworkPolicy to restrict traffic to your database Pods, but traffic still isn't being blocked. What's the most likely cause?**
The CNI plugin in use doesn't support/enforce NetworkPolicy — not all CNI plugins implement policy enforcement out of the box (this is a common gotcha, especially on some AWS VPC CNI configurations without additional policy enforcement enabled). Always confirm the CNI plugin's NetworkPolicy support before relying on it.

---

## Core Interview Q&A

**Q: What is a Pod, and why doesn't Kubernetes just run containers directly?**
A: A Pod is the smallest deployable unit in Kubernetes, wrapping one or more containers that share network and storage. Kubernetes uses Pods instead of raw containers because some containers must be tightly coupled — sharing localhost networking and volumes — and Pods give Kubernetes a consistent scheduling/scaling unit regardless of whether one or several containers are involved.

**Q: What's the relationship between a Deployment, a ReplicaSet, and a Pod?**
A: A Deployment manages ReplicaSets, and each ReplicaSet manages a set of identical Pods. The Deployment adds rolling update, rollback, and revision history capabilities on top of the ReplicaSet's basic job of maintaining a desired replica count.

**Q: When would you choose a StatefulSet over a Deployment?**
A: When Pods need stable, predictable network identity and their own persistent storage that follows them across rescheduling — typically for clustered stateful systems like databases or message queues where replica identity matters.

**Q: What's the difference between an Init Container and a sidecar container?**
A: An Init Container runs to completion before the main application container starts (sequential, one-time setup). A sidecar runs alongside the main container for the Pod's entire lifetime, providing an ongoing supporting function.

**Q: Why do we need a Service if every Pod already has its own IP?**
A: Pod IPs are not stable — Pods are frequently recreated (rollouts, crashes, rescheduling) and get a new IP each time. A Service provides one stable virtual IP/DNS name that automatically routes to whichever healthy Pods currently match its label selector.

**Q: What's the difference between ClusterIP, NodePort, and LoadBalancer?**
A: ClusterIP is internal-only cluster access (the default). NodePort opens a static port on every node for external access. LoadBalancer provisions an actual external cloud load balancer that routes into the cluster — the standard way to expose a service publicly on a cloud provider like AWS.

**Q: How does DNS-based service discovery work in Kubernetes?**
A: CoreDNS runs inside the cluster and automatically creates a DNS record for every Service (`<service>.<namespace>.svc.cluster.local`), letting Pods reach Services by name rather than hardcoded IP.

**Q: Is Kubernetes networking secure by default?**
A: No — by default all Pods can communicate with all other Pods with no restrictions. NetworkPolicies must be explicitly defined to restrict traffic, and they only take effect if the cluster's CNI plugin supports enforcing them.

**Q: What does kube-proxy actually do?**
A: It runs on every node and programs networking rules (via iptables or IPVS) that implement Service routing — directing traffic sent to a Service's virtual IP to one of the healthy backend Pods.
