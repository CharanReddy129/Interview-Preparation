# Kubernetes Notes — Part 4
## Scheduling & Resource Management | Health, Self-Healing & Rollouts | Security

---

## Scheduling & Resource Management

### Resource Requests vs Limits (CPU/Memory)
Every container in a Pod can declare two numbers per resource (CPU, memory):
- **Requests** — the amount of resource the scheduler **guarantees** is available for this container when deciding which node to place it on. The scheduler will only place a Pod on a node that has enough unreserved capacity to satisfy its requests.
- **Limits** — the maximum amount of resource a container is **allowed** to use at runtime, enforced by the kubelet/kernel.

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: my-app:1.0
      resources:
        requests:
          cpu: "250m"        # 0.25 CPU core
          memory: "256Mi"
        limits:
          cpu: "500m"        # 0.5 CPU core
          memory: "512Mi"
```

**What happens when limits are exceeded:**
- **CPU** — the container is throttled (slowed down), not killed. CPU is a compressible resource.
- **Memory** — the container is **OOMKilled** (exit code 137) if it exceeds its memory limit. Memory is not compressible — there's no "throttling" option, so the kernel kills the process instead.

**Interview framing:** "Requests affect scheduling decisions; limits affect runtime enforcement. If you set a request too low, you risk overcommitting a node; if you set a limit too low, you risk your own container getting throttled or OOMKilled."

### QoS Classes
Kubernetes automatically assigns a **Quality of Service (QoS) class** to every Pod based on how its requests/limits are set — this determines eviction priority under node resource pressure.

| QoS Class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | Every container has requests == limits, for both CPU and memory | Evicted last (highest protection) |
| **Burstable** | At least one container has requests set, but requests ≠ limits (or limits not set on all) | Evicted before Guaranteed, after BestEffort |
| **BestEffort** | No requests or limits set at all | Evicted first (lowest protection) |

**Interview framing:** "For critical production workloads (databases, core services), I'd set requests equal to limits to get Guaranteed QoS — accepting less scheduling flexibility in exchange for the pod being the last thing evicted under memory pressure."

### Node Selectors, Node Affinity/Anti-Affinity
- **Node Selector** — the simplest way to constrain a Pod to nodes with a specific label; a flat key-value match, no complex logic.
```yaml
spec:
  nodeSelector:
    disktype: ssd
```
- **Node Affinity** — a more expressive version of nodeSelector, supporting `requiredDuringSchedulingIgnoredDuringExecution` (hard rule — must be satisfied) and `preferredDuringSchedulingIgnoredDuringExecution` (soft rule — scheduler tries but won't fail if unmet), plus richer operators (`In`, `NotIn`, `Exists`, etc.).
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
```
- **Node Anti-Affinity** — achieved using the `NotIn`/`DoesNotExist` operators within node affinity rules (there's no separate "anti-affinity" field for nodes — it's expressed via the same nodeAffinity mechanism).

### Taints and Tolerations
**Taints** are applied to **nodes** to repel Pods that don't explicitly tolerate them — the inverse of affinity (which pulls Pods toward nodes; taints push Pods away unless they opt in).
```bash
kubectl taint nodes node1 key=value:NoSchedule
```
**Tolerations** are applied to **Pods**, allowing (but not forcing) them to be scheduled onto nodes with a matching taint.
```yaml
spec:
  tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
```
Taint effects:
| Effect | Behavior |
|---|---|
| `NoSchedule` | New Pods without a matching toleration won't be scheduled onto the node |
| `PreferNoSchedule` | Scheduler tries to avoid it, but isn't guaranteed |
| `NoExecute` | New Pods won't be scheduled, AND existing Pods without the toleration are evicted from the node |

**Common real-world use case:** dedicating specific nodes to specific workloads (e.g., GPU nodes tainted so only GPU-requiring Pods with the matching toleration land there), or marking nodes for maintenance.

**Interview framing — Affinity vs Taints/Tolerations:** "Affinity is about a Pod expressing a preference to go toward certain nodes. Taints/tolerations are about a node actively repelling Pods unless they explicitly opt in — they solve different problems and are often used together (e.g., a taint to reserve a node pool, plus affinity to actively steer the right Pods there)."

### Pod Affinity/Anti-Affinity
While node affinity is about Pod-to-node placement, **Pod affinity/anti-affinity** is about Pod-to-Pod placement relative to each other.
- **Pod Affinity** — schedule a Pod close to (same node/zone as) other Pods matching a label selector. Use case: co-locating a cache with the app that heavily uses it, to reduce network latency.
- **Pod Anti-Affinity** — schedule a Pod away from other Pods matching a label selector. Use case: spreading replicas of the same Deployment across different nodes/zones for high availability, so a single node failure doesn't take down all replicas at once.
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["my-app"]
          topologyKey: "kubernetes.io/hostname"
```

### Horizontal Pod Autoscaler (HPA)
The **HPA** automatically scales the **number of Pod replicas** in a Deployment/ReplicaSet/StatefulSet up or down based on observed metrics (CPU/memory utilization by default, or custom/external metrics via the Metrics API).

**Example:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```
Requires the **Metrics Server** to be running in the cluster to supply the resource usage data HPA reads.

### Vertical Pod Autoscaler (VPA) — Conceptual
Where HPA changes the **number** of replicas, **VPA** adjusts the **resource requests/limits** of existing Pods based on observed usage over time — e.g., automatically bumping a container's memory request if it's consistently using more than initially specified.

**Important caveat:** VPA typically needs to **recreate the Pod** to apply new resource values (you can't resize a running container's limits in place in most setups), so it causes disruption — and **VPA and HPA should generally not both target CPU/memory on the same workload simultaneously**, as they can conflict (fight over scaling dimensions). VPA is more commonly run in "recommendation-only" mode in practice.

### Cluster Autoscaler (Especially Relevant with EKS Node Groups)
While HPA/VPA scale Pods, the **Cluster Autoscaler** scales the **nodes themselves** — adding new nodes when Pods are unschedulable due to insufficient cluster capacity, and removing underutilized nodes when they're no longer needed (as long as Pods on them can be safely rescheduled elsewhere).

On EKS specifically, Cluster Autoscaler works by adjusting the desired size of an **EKS Node Group** (an underlying AWS Auto Scaling Group). It watches for `Pending` Pods that can't be scheduled due to insufficient resources, and scales the ASG up in response.

**Cluster Autoscaler vs Karpenter (EKS-specific, worth knowing):** Karpenter is a newer, more flexible node-provisioning alternative built specifically to address Cluster Autoscaler's limitations — it provisions right-sized nodes directly (not limited to pre-defined node group instance types) and can react faster. Increasingly common in modern EKS setups, and a good thing to mention as awareness even if you've primarily used Cluster Autoscaler.

---

## Health, Self-Healing & Rollouts

### Liveness Probe vs Readiness Probe vs Startup Probe
Kubernetes uses **probes** to determine container health, and each answers a different question:

| Probe | Question it answers | Action on failure |
|---|---|---|
| **Liveness Probe** | "Is this container still alive/functioning?" | Kubernetes **restarts** the container |
| **Readiness Probe** | "Is this container ready to receive traffic right now?" | Kubernetes **removes the Pod from Service endpoints** (stops routing traffic to it) — does NOT restart it |
| **Startup Probe** | "Has this container finished starting up yet?" | Kubernetes waits before running liveness/readiness checks, preventing a slow-starting app from being killed prematurely by the liveness probe |

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probes-demo
spec:
  containers:
    - name: app
      image: my-app:1.0
      startupProbe:
        httpGet:
          path: /health
          port: 8080
        failureThreshold: 30
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
```

**Interview framing — classic trap question:** "Why would a Pod be `Running` but still not receiving any traffic?" — its readiness probe is failing, so it's been removed from the Service's endpoint list, even though the container itself hasn't crashed and liveness is fine.

### Rolling Updates and Rollback Strategies
Covered at a high level in Part 2 (Deployments) — recapped here with the operational commands:
- Trigger a rollout: change the Pod template (e.g., `kubectl set image deployment/my-app app=my-app:2.0`) or `kubectl apply` an updated manifest.
- Monitor: `kubectl rollout status deployment/my-app`.
- Roll back: `kubectl rollout undo deployment/my-app` (to previous revision) or `kubectl rollout undo deployment/my-app --to-revision=N` (to a specific one).
- View history: `kubectl rollout history deployment/my-app`.

### Deployment Strategies
| Strategy | How it works | Tradeoff |
|---|---|---|
| **Recreate** | Terminates all old Pods first, then creates all new ones | Simple, but causes downtime — used when old and new versions can't safely run simultaneously (e.g., incompatible schema) |
| **RollingUpdate** (Deployment default) | Gradually replaces old Pods with new ones, controlled by `maxSurge`/`maxUnavailable` | Zero-downtime, but both versions run simultaneously for a period — requires backward compatibility |
| **Blue-Green** (conceptual — not a native K8s primitive) | Deploy the new version entirely alongside the old one (two full environments), then switch traffic all at once by updating a Service selector or Ingress | Instant cutover and instant rollback (just switch back), but requires double the resources temporarily |
| **Canary** (conceptual — not a native K8s primitive) | Route a small percentage of traffic to the new version first, monitor, then gradually increase | Lowest risk — catches problems with minimal blast radius — but needs more sophisticated traffic-splitting (Ingress annotations, service mesh, or tools like Argo Rollouts) |

**Interview framing:** "Kubernetes natively supports Recreate and RollingUpdate out of the box via Deployment strategy. Blue-Green and Canary aren't native Deployment strategies — they're patterns you implement using Services/Ingress traffic-splitting, or more robustly with a tool like Argo Rollouts, which is exactly why teams doing GitOps with Argo CD often pair it with Argo Rollouts for progressive delivery."

### Pod Disruption Budgets (PDB)
A **PodDisruptionBudget** limits how many Pods of a given set can be **voluntarily disrupted** at once (e.g., during a node drain for maintenance, or a Cluster Autoscaler scale-down) — protecting availability during planned operations. It does NOT protect against involuntary disruptions like a node crashing unexpectedly.

**Example:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2          # or use maxUnavailable instead
  selector:
    matchLabels:
      app: my-app
```

**Interview framing:** "PDBs matter most during planned maintenance — if I'm draining a node for an EKS upgrade, the PDB ensures the drain won't proceed in a way that would take my Deployment below its minimum available replica count."

---

## 8. Security

### RBAC (Role-Based Access Control)
RBAC controls **who can do what** within the cluster, using four core objects:
- **Role** — a set of permissions (verbs like `get`/`list`/`create`/`delete` on specific resources), scoped to a **single namespace**.
- **ClusterRole** — the same idea, but scoped **cluster-wide** (or for cluster-scoped resources like Nodes, which don't belong to any namespace).
- **RoleBinding** — grants the permissions in a Role (or ClusterRole) to a specific user/group/ServiceAccount, **within a specific namespace**.
- **ClusterRoleBinding** — grants the permissions in a ClusterRole to a user/group/ServiceAccount **cluster-wide**.

**Example — Role + RoleBinding (namespace-scoped):**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Interview framing — a common trap:** "A ClusterRole bound via a RoleBinding (not a ClusterRoleBinding) only grants those permissions within that RoleBinding's namespace, even though the ClusterRole itself is cluster-scoped in definition — this is actually a useful, common pattern for reusing one ClusterRole definition across many namespace-scoped bindings."

### ServiceAccounts
A **ServiceAccount** provides an identity for **processes running inside Pods** to authenticate to the Kubernetes API server (as opposed to User accounts, which are for humans and aren't managed as Kubernetes objects at all). Every namespace has a `default` ServiceAccount automatically, but best practice is to create dedicated, minimally-privileged ServiceAccounts per application.

**Example:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: dev
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app-sa
      containers:
        - name: my-app
          image: my-app:1.0
```

### Namespaces
A **Namespace** is a mechanism for dividing a single cluster into multiple **virtual clusters** — providing scope for names (two Pods can share the same name in different namespaces) and a boundary for applying RBAC, resource quotas, and network policies.

**Common use cases:** separating environments (`dev`, `staging`, `prod`) or teams within a shared cluster. Note: namespaces provide a *logical* boundary, not full hard multi-tenancy security isolation by themselves — that requires combining namespaces with RBAC, NetworkPolicies, and resource quotas together.

**Example:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

### Pod Security Standards / Admission
**Pod Security Standards** define three tiers of security restriction that can be enforced on Pods within a namespace, via the built-in **Pod Security Admission** controller:

| Level | Restriction |
|---|---|
| `privileged` | No restrictions — allows known privilege escalations (e.g., running as root, host namespace access) |
| `baseline` | Blocks known privilege escalations, but stays broadly compatible with common workloads |
| `restricted` | Heavily locked down — enforces current Pod hardening best practices (non-root, no privilege escalation, dropped capabilities, etc.) |

Applied via a namespace label:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

### Network Policies (Security Angle, Revisited)
As covered in Part 2 — worth re-emphasizing in a security context: Kubernetes networking is open by default, so NetworkPolicies are a foundational **defense-in-depth** control, not an optional nice-to-have. A common production baseline is a **default-deny policy** per namespace, with explicit allow rules layered on top for legitimate traffic — rather than starting open and trying to lock down individual paths after the fact.

**Example — default-deny all ingress in a namespace:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### Secrets Management Best Practices
Recapping and extending from Part 3: native Kubernetes Secrets are only base64-encoded, not encrypted, by default. Production-grade practice layers on:
- **etcd encryption at rest**
- **Strict RBAC** limiting Secret read access
- **External Secrets Operator** — syncs real secrets from an external source (AWS Secrets Manager, HashiCorp Vault) into Kubernetes Secrets at runtime, so the actual sensitive values live in a purpose-built secrets manager with proper rotation/auditing, not natively in etcd.

### IRSA (IAM Roles for Service Accounts) — EKS-Specific
**IRSA** lets you associate a specific **AWS IAM Role** with a **Kubernetes ServiceAccount**, so Pods using that ServiceAccount can call AWS APIs (S3, Secrets Manager, DynamoDB, etc.) with fine-grained, least-privilege IAM permissions — **without** needing to distribute static AWS access keys into the cluster.

**How it works (conceptually):** EKS runs an OIDC identity provider; when a Pod using an annotated ServiceAccount makes an AWS API call via the AWS SDK, it exchanges a Kubernetes-issued service account token for temporary AWS credentials tied to the associated IAM role, via AWS STS.

**Example:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app-sa    # Pods using this SA assume the IAM role
      containers:
        - name: my-app
          image: my-app:1.0
```

**Interview framing:** "IRSA is the reason we don't hardcode AWS credentials in Secrets or environment variables at all for AWS API access from within EKS — it's the standard, least-privilege way to grant a specific workload exactly the AWS permissions it needs, scoped per ServiceAccount rather than per node."

---

## Scenario-Based Questions

**Q1: A Pod keeps getting OOMKilled even though the node has plenty of free memory. What's the likely cause?**
The container's memory **limit** is set too low for its actual usage — OOMKilled is triggered by exceeding the container's own limit, not by node-wide memory pressure. Check `kubectl describe pod` for the limit value and compare it against actual usage (`kubectl top pod`), then raise the limit or investigate a memory leak if usage is unexpectedly high.

**Q2: You need to guarantee that no more than one replica of a 5-replica Deployment is ever down at a time during a planned node drain for cluster upgrades. What do you configure?**
A PodDisruptionBudget with `maxUnavailable: 1` (or `minAvailable: 4`) — this ensures `kubectl drain` respects that constraint and won't evict Pods beyond what the budget allows during voluntary disruptions.

**Q3: A Pod shows status `Running` in `kubectl get pods`, but it's receiving zero traffic from its Service. What do you check?**
The readiness probe — a Pod can be alive and Running while consistently failing its readiness check, which removes it from the Service's endpoint list without restarting it. Check `kubectl describe pod` for readiness probe failures and `kubectl get endpoints` to confirm the Pod isn't listed as a backend.

**Q4: Your team wants to reserve a set of GPU nodes exclusively for ML workloads, without regular workloads accidentally landing there and without ML Pods being scheduled anywhere else. What do you configure?**
Taint the GPU nodes (e.g., `nvidia.com/gpu=true:NoSchedule`) so regular Pods are repelled by default, add matching tolerations to the ML Pods so they're allowed on those nodes, and additionally use node affinity/nodeSelector on the ML Pods so they're actively steered there (taints alone only repel — they don't attract).

**Q5: Cluster Autoscaler isn't adding new nodes even though several Pods are stuck in `Pending` due to insufficient resources. What would you check?**
Whether the Pending Pods' resource requests can actually be satisfied by any instance type available in the configured Node Group/Auto Scaling Group (Cluster Autoscaler won't scale up if no available node type could ever fit the Pod), whether the ASG has hit its configured max size, and whether there are unmet scheduling constraints (node affinity, taints without matching tolerations) that Cluster Autoscaler also has to account for before adding a node that would actually help.

**Q6: A developer's Pod can list Secrets in the `dev` namespace but shouldn't be able to touch Secrets in `prod`. How is this enforced?**
Via namespace-scoped RBAC — grant a Role (not ClusterRole) with `get`/`list` on Secrets, bound via a RoleBinding scoped specifically to the `dev` namespace. As long as no ClusterRoleBinding or `prod`-namespace RoleBinding grants equivalent access, the developer's permissions won't extend to `prod`.

**Q7: A Pod running in EKS needs to read objects from an S3 bucket. What's the recommended way to grant this access, and what should you avoid?**
Use IRSA — create an IAM role with least-privilege S3 permissions, associate it with a dedicated ServiceAccount via the `eks.amazonaws.com/role-arn` annotation, and have the Pod use that ServiceAccount. Avoid hardcoding AWS access keys as Kubernetes Secrets or environment variables, since that bypasses IAM's temporary-credential model and creates long-lived, harder-to-rotate credentials.

---

## Core Interview Q&A

**Q: What's the difference between a resource request and a resource limit?**
A: A request is what the scheduler guarantees is available on a node before placing a Pod there. A limit is the maximum a container is allowed to consume at runtime — exceeding a CPU limit throttles the container, while exceeding a memory limit gets it OOMKilled.

**Q: What determines a Pod's QoS class?**
A: Whether its containers have requests/limits set, and whether they're equal. Guaranteed requires requests == limits for every resource on every container; Burstable is a partial match; BestEffort is no requests/limits at all — this ordering also determines eviction priority under node pressure.

**Q: What's the difference between taints/tolerations and node affinity?**
A: Taints/tolerations are node-centric — a taint repels Pods unless they explicitly tolerate it. Node affinity is Pod-centric — it expresses a Pod's preference or requirement to land on certain nodes. They're often combined: a taint to reserve capacity, plus affinity to actively steer the intended Pods there.

**Q: What's the difference between HPA, VPA, and Cluster Autoscaler?**
A: HPA scales the number of Pod replicas based on metrics. VPA adjusts a Pod's resource requests/limits based on observed usage (typically requiring a Pod recreate). Cluster Autoscaler scales the number of nodes in the cluster itself, based on whether Pods are unschedulable due to insufficient capacity.

**Q: Explain the difference between a liveness probe and a readiness probe.**
A: A liveness probe determines whether a container should be restarted because it's no longer functioning. A readiness probe determines whether a container should currently receive traffic — failing it removes the Pod from Service endpoints without restarting the container.

**Q: What does a Pod Disruption Budget protect against?**
A: Voluntary disruptions — like node drains during maintenance or Cluster Autoscaler scale-downs — by limiting how many Pods from a given set can be evicted at once. It does not protect against involuntary disruptions like a node crashing.

**Q: What's the difference between a Role and a ClusterRole?**
A: A Role's permissions are scoped to a single namespace. A ClusterRole's permissions are scoped cluster-wide (or apply to cluster-scoped resources) — but a ClusterRole can still be bound to a single namespace's worth of access if referenced by a RoleBinding rather than a ClusterRoleBinding.

**Q: What problem does IRSA solve on EKS?**
A: It lets Pods assume specific, least-privilege IAM roles (via their ServiceAccount) to call AWS APIs, without distributing long-lived AWS access keys into the cluster — credentials are temporary and scoped per-workload rather than per-node.
