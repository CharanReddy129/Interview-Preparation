# Kubernetes Notes - Part 5
## Troubleshooting & Real-World Scenarios | Kubernetes on AWS / EKS-Specific | Advanced/Extensibility

---

## Troubleshooting & Real-World Scenarios

This section is less about definitions and more about **diagnostic muscle memory** — interviewers use these as scenario prompts constantly, since they reveal whether you've actually operated a cluster or just read about one. The general debugging sequence to always default to:
```bash
kubectl get pods -o wide                  # quick status + node placement
kubectl describe pod <pod>                 # events, conditions, container states — start here
kubectl logs <pod> [-c <container>] [--previous]   # application-level errors
kubectl get events --sort-by='.lastTimestamp'      # cluster-wide recent events
```

### Pod Stuck in `Pending`
**What it means:** the Pod has been accepted by the API server but hasn't been scheduled onto any node yet.

**Common causes:**
- **Insufficient cluster resources** — no node has enough unreserved CPU/memory to satisfy the Pod's requests.
- **Unsatisfiable scheduling constraints** — node affinity/nodeSelector referencing a label no node has, or taints on all candidate nodes with no matching toleration on the Pod.
- **PersistentVolumeClaim not bound** — if the Pod references a PVC that hasn't been provisioned/bound yet (e.g., StorageClass misconfiguration), the Pod can't be scheduled.
- **Cluster Autoscaler hasn't caught up yet** — if scaling out is needed, there's a natural delay before a new node joins and becomes schedulable.

**Diagnosis:**
```bash
kubectl describe pod <pod>     # check the Events section at the bottom — it names the exact reason
```
The `describe` output almost always states the reason explicitly (e.g., `0/5 nodes are available: 5 Insufficient memory`).

### Pod in `CrashLoopBackOff`
**What it means:** the container starts, exits (crashes), and Kubernetes keeps restarting it per the restart policy — with an exponential backoff delay between attempts (hence the name).

**Common causes:**
- Application-level error on startup (missing config, bad env var, unhandled exception) — check `kubectl logs <pod> --previous` (the `--previous` flag is essential here, since the current container instance may not have logged anything yet).
- Failing liveness probe repeatedly restarting an otherwise-working container (misconfigured probe path/port/timing).
- Missing dependency the app can't tolerate at startup (e.g., can't reach a database and doesn't retry gracefully) — often fixable with an Init Container that waits for the dependency first.
- Container's main process exits immediately (e.g., wrong `CMD`/`ENTRYPOINT`, or the process finished and there's nothing left running).

**Diagnosis:**
```bash
kubectl logs <pod> --previous          # logs from the crashed instance, not the current restart attempt
kubectl describe pod <pod>              # check exit code and "Last State: Terminated" reason
```

### Pod Stuck in `ImagePullBackOff`
**What it means:** the kubelet can't pull the container image specified in the Pod spec, and is backing off between retry attempts.

**Common causes:**
- Typo in the image name/tag, or the tag genuinely doesn't exist in the registry.
- Private registry requiring authentication, with no (or an incorrect) `imagePullSecrets` configured on the Pod/ServiceAccount.
- Registry rate limiting (common with anonymous Docker Hub pulls in CI-heavy environments).
- Network/DNS issue preventing the node from reaching the registry at all (e.g., missing NAT gateway route in a private subnet — relevant on EKS).

**Diagnosis:**
```bash
kubectl describe pod <pod>     # Events section shows the exact pull error message
```

**Fix (private registry auth example):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  imagePullSecrets:
    - name: my-registry-secret
  containers:
    - name: app
      image: my-private-registry.com/my-app:1.0
```

### Service Not Routing Traffic to Pods
**What it means:** requests to a Service aren't reaching any backend Pod, even though the Service object exists.

**Common causes:**
- **Label selector mismatch** — the Service's `selector` doesn't actually match the labels on the target Pods (the single most common cause — always check this first).
- **Pods failing their readiness probe** — they're `Running` but not in the Service's Endpoints list, since only Ready Pods are included as backends.
- **Wrong `targetPort`** — the Service points to a container port the app isn't actually listening on.
- App inside the container is bound to `127.0.0.1` instead of `0.0.0.0`, making it unreachable from outside the container's own network namespace even though the process is running fine.

**Diagnosis:**
```bash
kubectl get endpoints <service-name>          # if this is empty, no Pods currently match/are ready
kubectl describe service <service-name>       # confirm the Selector matches your Pods' actual labels
kubectl get pods --show-labels                # compare against what the Service expects
```

### Node `NotReady`
**What it means:** the node's kubelet has stopped reporting healthy status to the control plane (or has stopped reporting at all) within the expected heartbeat window.

**Common causes:**
- kubelet process crashed or isn't running on the node.
- Node resource exhaustion (disk pressure, memory pressure) triggering kubelet-level health conditions.
- Network partition between the node and the control plane (security group misconfiguration on EKS is a classic culprit).
- Container runtime (containerd) unhealthy or unresponsive on the node.

**Diagnosis:**
```bash
kubectl describe node <node-name>     # check Conditions section (MemoryPressure, DiskPressure, Ready status + reason)
kubectl get nodes -o wide
```
On EKS specifically, also worth checking: the node's underlying EC2 instance status checks in the AWS console, and whether the node's IAM role/security group still allows it to communicate with the control plane.

### OOMKilled Pods
**What it means:** a container exceeded its memory **limit** and the kernel's OOM killer terminated it — visible as exit code **137** and reason `OOMKilled` in `kubectl describe pod`.

**Diagnosis and fix:**
```bash
kubectl describe pod <pod>          # confirm "Last State: Terminated, Reason: OOMKilled"
kubectl top pod <pod>                # check actual current usage vs configured limit
```
- If usage is expected/legitimate — raise the memory `limit` (and likely `request`) to a realistic value based on observed usage.
- If usage looks like an unbounded leak — that's an application-level issue (needs code-level investigation, not just a bigger limit).
- Remember: this is entirely about the **container's own limit**, not overall node memory pressure — a Pod can get OOMKilled even on a node with plenty of free memory if its own limit is too low.

### Debugging DNS Resolution Issues Inside the Cluster
**What it means:** a Pod can't resolve another Service's DNS name (`my-service.namespace.svc.cluster.local` or its short form).

**Common causes:**
- **CoreDNS Pods themselves are unhealthy** — check them directly, since if CoreDNS is down, *nothing* in the cluster can resolve names.
- **Namespace scoping mistake** — using the short name (`my-service`) from a Pod in a *different* namespace than the Service (short names only resolve within the same namespace).
- **NetworkPolicy blocking egress to CoreDNS** — an overly strict default-deny NetworkPolicy that doesn't explicitly allow DNS traffic (UDP/TCP port 53) will silently break resolution for affected Pods.
- **`/etc/resolv.conf` misconfiguration** inside the Pod (rare, usually only relevant with custom `dnsPolicy` settings).

**Diagnosis:**
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns     # check CoreDNS Pods are Running/healthy
kubectl exec -it <pod> -- nslookup my-service.my-namespace.svc.cluster.local
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl logs -n kube-system -l k8s-app=kube-dns          # check CoreDNS's own logs for errors
```

**Interview framing:** "DNS troubleshooting always starts the same way for me — first confirm CoreDNS itself is healthy, since everything else downstream is meaningless if the DNS server is the actual problem, then work through namespace scoping and NetworkPolicy egress rules for port 53."

---

## 13. Kubernetes on AWS / EKS-Specific

### EKS Control Plane vs Self-Managed vs Fargate Node Groups
- **EKS Control Plane** — fully managed by AWS: the API server, etcd, scheduler, and controller manager run across multiple AWS accounts/AZs for high availability, with AWS handling patching, scaling, and backups of the control plane itself. You never directly access or manage these components — you only ever manage worker nodes and workloads.
- **Self-managed node groups** — you provision and manage the underlying EC2 instances yourself (AMI selection, patching, scaling via your own Auto Scaling Group configuration) and join them to the cluster manually or via bootstrap scripts. Maximum control, maximum operational overhead.
- **Managed node groups** — AWS provisions and manages the EC2 instances' lifecycle (creation, updates, graceful draining/replacement during upgrades) on your behalf, while you still choose instance types, scaling config, and AMI type. The common middle-ground choice for most teams.
- **Fargate node groups** — fully serverless: there are no EC2 instances to manage at all. Each Pod gets its own dedicated, right-sized compute automatically provisioned by AWS. No node-level access, no DaemonSets (since there's no persistent "node" to run one on), and some networking/storage limitations (e.g., certain CNI/CSI features rely on node-level agents that don't apply under Fargate).

**Interview framing:** "The tradeoff ladder is roughly: self-managed gives you full control at full operational cost, managed node groups offload node lifecycle management while keeping EC2-level flexibility, and Fargate removes node management entirely at the cost of some flexibility (no DaemonSets, some CNI/CSI limitations) — the right choice depends on how much of that operational surface the team wants to own."

### IRSA in Depth
(Recapping and extending from Part 4's Security section, since it's central to EKS interviews.)

**IAM Roles for Service Accounts (IRSA)** lets a Kubernetes ServiceAccount assume a specific AWS IAM role, so Pods using that ServiceAccount get temporary, scoped AWS credentials — without static access keys anywhere in the cluster.

**How it actually works, step by step:**
1. EKS clusters have an associated **OIDC identity provider** (an IAM OIDC provider registered against the cluster's own OIDC issuer URL).
2. You create an IAM role with a **trust policy** that trusts this OIDC provider, scoped to a specific Kubernetes namespace + ServiceAccount name (via a condition on the `sub` claim).
3. The ServiceAccount is annotated with the IAM role's ARN (`eks.amazonaws.com/role-arn`).
4. When a Pod using that ServiceAccount starts, EKS's Pod Identity webhook automatically injects a projected, time-limited **service account token** and sets AWS SDK environment variables (`AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`).
5. The AWS SDK inside the container automatically exchanges that token for temporary AWS credentials via **AWS STS's `AssumeRoleWithWebIdentity`** call — no code changes needed if using a current AWS SDK version.

**Example (recap from Part 4):**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
```

**Interview framing:** "The key mechanism to name explicitly is `AssumeRoleWithWebIdentity` via STS — that's the actual AWS API call happening behind the scenes, and being able to name it signals real understanding versus just knowing IRSA exists as a concept."

**Note:** **EKS Pod Identity** is a newer, simpler alternative to IRSA (introduced more recently) that removes the need to manage OIDC trust policies per role — worth mentioning as awareness if asked "is there anything newer than IRSA," since AWS has been pushing teams toward it, though IRSA remains extremely widely deployed and a safe thing to know deeply.

### AWS Load Balancer Controller (ALB/NLB Ingress)
The **AWS Load Balancer Controller** is a controller you install into an EKS cluster that watches for `Ingress` and `Service` (type `LoadBalancer`) objects and provisions real AWS load balancers accordingly:
- **Ingress objects** → provisions an **Application Load Balancer (ALB)** — Layer 7, HTTP/HTTPS routing, path/host-based rules, TLS termination.
- **Service type `LoadBalancer`** (with the right annotation) → provisions a **Network Load Balancer (NLB)** — Layer 4, for TCP/UDP traffic, extremely high throughput/low latency use cases.

**Example — Ingress provisioning an ALB:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```
`target-type: ip` routes ALB traffic directly to Pod IPs (works cleanly with the AWS VPC CNI's real-VPC-IP-per-Pod model) rather than through NodePort — the modern recommended approach on EKS.

### EKS Networking: VPC CNI, Subnets, Security Groups per Pod
Recapping and extending from Part 2's CNI discussion:
- The **AWS VPC CNI** assigns each Pod a **real IP address from the VPC's subnet range** (not an overlay network) — meaning Pods are natively routable within the VPC and can be reached/secured using standard AWS networking primitives.
- **Practical implication — IP exhaustion:** because every Pod consumes a real VPC IP, node capacity for Pods can be constrained by the number of ENI (Elastic Network Interface) IP addresses available per instance type, not just CPU/memory — a real, sometimes-surprising scaling limit worth knowing about (mitigated by using larger instance types, IPv6, or "prefix delegation" mode in the CNI for higher IP density per ENI).
- **Security Groups for Pods** — a VPC CNI feature that lets you attach specific AWS Security Groups directly to individual Pods (via `SecurityGroupPolicy` objects matching Pod labels), rather than only being able to secure traffic at the node level — useful when different Pods on the same node need different AWS-level network access controls.
- **Subnet planning matters** — private subnets for worker nodes (standard practice), with a NAT Gateway for outbound internet access (image pulls, etc.), and adequately sized CIDR ranges since every Pod (not just every node) consumes an IP.

### EKS Upgrades and Version Skew Policy
Kubernetes only officially supports a limited version skew between components:
- **kubelet vs control plane (API server):** the kubelet on a worker node can be **up to 2 minor versions older** than the control plane's API server version (e.g., control plane on 1.29, nodes as old as 1.27 are still supported) — this is what allows a **rolling, gradual upgrade** across a large fleet of nodes rather than requiring everything to update in lockstep.
- **Recommended real-world upgrade order:** upgrade the EKS control plane first (AWS handles this with minimal disruption), then upgrade node groups (managed node groups can do this with a rolling replacement), then upgrade workloads/add-ons (CNI, CoreDNS, kube-proxy versions need to stay compatible with the new control plane version too — AWS publishes a compatibility matrix for these).
- **EKS only supports upgrading one minor version at a time** (e.g., 1.28 → 1.29, not 1.28 → 1.30 directly) — a very practical, commonly-tested detail for anyone who's actually run an upgrade.

**Interview framing:** "The version skew policy is exactly what makes rolling upgrades possible without downtime — you're never forced to upgrade every node simultaneously, since the control plane tolerates nodes running up to two minor versions behind while you work through the fleet."

### Cluster Autoscaler vs Karpenter
(Recapping and extending from Part 4.)

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| **How it scales** | Adjusts the desired size of pre-defined Node Groups/Auto Scaling Groups | Provisions individual EC2 instances directly, choosing instance type/size dynamically based on actual pending Pod requirements |
| **Instance type flexibility** | Limited to whatever instance types are configured in the target Node Group(s) | Can select from a broad range of instance types/families on the fly to best-fit the actual workload, often improving bin-packing and cost |
| **Speed** | Somewhat slower — bound by ASG scaling mechanics | Generally faster to provision and more responsive to sudden demand spikes |
| **Setup complexity** | Simpler, well-established, tightly tied to traditional Node Group configuration | More modern, more flexible, but a newer tool with its own configuration model (`NodePool`/`EC2NodeClass` CRDs) to learn |

**Interview framing:** "Karpenter is the direction most new EKS setups are heading for its faster, more cost-efficient bin-packing, but Cluster Autoscaler is still extremely common in existing production environments, and it's worth being able to speak to both rather than assuming one has fully replaced the other yet."

---

## 14. Advanced/Extensibility (Good to Know Conceptually)

### Custom Resource Definitions (CRDs)
A **CustomResourceDefinition** lets you extend the Kubernetes API with your own object types, beyond the built-ins (Pod, Deployment, Service, etc.) — once a CRD is registered, you can create, get, list, and watch instances of your custom resource exactly like any native Kubernetes object, using the same `kubectl` and API mechanics.

**Example — defining a CRD:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
                size:
                  type: string
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
```
**Using it once registered:**
```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-db
spec:
  engine: postgres
  size: "10Gi"
```
On its own, a CRD is just a new **schema/API type** — creating a `Database` object above does nothing by itself unless something is actually watching for it and taking action. That "something" is an **Operator**.

### Operators (Operator Pattern)
An **Operator** is a controller (custom code, typically built with frameworks like Kubebuilder or the Operator SDK) that watches a CRD's custom resources and implements **domain-specific operational logic** to reconcile them — essentially encoding human operational expertise into software, the same reconciliation-loop pattern Kubernetes itself uses internally, extended to manage things Kubernetes doesn't understand natively.

**Real-world example:** a Postgres Operator watching `Database` custom resources (like the CRD above) would, upon seeing a new `Database` object, actually provision a StatefulSet, PVC, Service, and handle backup scheduling, failover, and version upgrades for that specific database — all the operational knowledge a human DBA would apply, automated and driven declaratively.

**Interview framing:** "A CRD is just the API extension — the shape of a new kind of object. An Operator is the actual controller logic that makes that object meaningful, watching for changes and taking real action to reconcile the cluster toward what the custom resource declares."

### Admission Controllers (Mutating/Validating Webhooks)
**Admission Controllers** intercept requests to the Kubernetes API server **after authentication/authorization but before the object is persisted to etcd**, letting you enforce policy or modify objects on the fly.

- **Mutating Admission Webhooks** — can modify the object before it's persisted (e.g., automatically injecting a sidecar container into every Pod matching certain labels — this is exactly how service mesh sidecar injection, like Istio's Envoy proxy, actually works under the hood).
- **Validating Admission Webhooks** — can only accept or reject the object (no modification), used to enforce policy (e.g., "reject any Pod that doesn't set resource requests/limits," or "reject any image not from an approved registry").

**Example — a ValidatingWebhookConfiguration (conceptual shape):**
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: require-resource-limits
webhooks:
  - name: require-resource-limits.example.com
    clientConfig:
      service:
        name: policy-webhook-service
        namespace: policy-system
        path: "/validate"
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
    failurePolicy: Fail
```
**Real-world tools built on this mechanism:** OPA Gatekeeper and Kyverno are the two most common policy engines used to author these validation/mutation rules declaratively rather than writing a custom webhook server from scratch — worth naming both if this comes up, since hand-rolling webhook servers is relatively rare compared to using one of these frameworks.

**Interview framing:** "Order matters here — mutating webhooks run before validating webhooks, so a mutation can still be checked by policy afterward. This whole mechanism is also exactly how Pod Security Admission (from Part 4) is implemented internally — it's a built-in validating admission controller."

### Multi-Cluster / Multi-Tenancy Concepts (Awareness Level)
- **Multi-tenancy within a single cluster** — achieved via a combination of Namespaces (logical isolation), RBAC (access control), ResourceQuotas/LimitRanges (resource fairness between tenants), and NetworkPolicies (traffic isolation) — "soft" multi-tenancy, since tenants still share the same control plane and node kernel.
- **Hard multi-tenancy** — genuinely isolating tenants at a stronger boundary, typically via **separate clusters per tenant** (or per environment/team), since Kubernetes namespaces alone were never designed as a hard security boundary against a sufficiently privileged or malicious co-tenant.
- **Multi-cluster management tools** — worth knowing by name even at a conceptual level: **Karmada** and **Cluster API** (for provisioning/managing the lifecycle of multiple clusters declaratively), and service-mesh-based approaches (e.g., Istio multi-cluster) for connecting workloads across cluster boundaries.
- **Why organizations go multi-cluster:** blast-radius containment (one cluster's issue doesn't take down everything), regulatory/data-residency requirements (different regions/clouds), hard tenant isolation, or simply exceeding a single cluster's practical scaling limits.

**Interview framing:** "For entry-level roles, the main thing worth being able to articulate is why namespaces alone aren't real security isolation, and that when hard isolation is actually required, the practical answer is usually separate clusters rather than trying to harden namespace boundaries indefinitely."

---

## Scenario-Based Questions

**Q1: A new Deployment's Pods are all stuck in `Pending`, and `kubectl describe pod` shows "0/8 nodes are available: 8 Insufficient cpu." What are your options to resolve this, beyond just waiting?**
Either reduce the Pod's CPU request if it's over-provisioned relative to actual need, or scale the cluster's capacity — if Cluster Autoscaler/Karpenter is configured, this should self-resolve once it detects the unschedulable Pods, but if not (or if it's hit its max size), manually scaling the node group or increasing its max size is the direct fix.

**Q2: You deploy a new image version and Pods immediately go into `CrashLoopBackOff`. `kubectl logs` on the current Pod shows nothing. What's your next command, and why?**
`kubectl logs <pod> --previous` — since the container likely crashed and restarted already by the time you ran the first `logs` command, and `--previous` retrieves logs from the last terminated instance rather than the fresh (and likely still-empty) current one.

**Q3: Your app's Pods are healthy and passing all probes, but external users report intermittent connection failures. You suspect DNS. Walk through your diagnostic sequence.**
First confirm CoreDNS Pods are healthy (`kubectl get pods -n kube-system -l k8s-app=kube-dns`), then test resolution directly from an affected Pod (`kubectl exec -it <pod> -- nslookup <service>.<namespace>.svc.cluster.local`), then check for any NetworkPolicy that might be blocking egress on port 53 to CoreDNS, since a default-deny policy without an explicit DNS allow rule is a common, easy-to-miss cause of intermittent-looking failures.

**Q4: You're planning an EKS upgrade from 1.27 to 1.30. What's wrong with doing this in a single step, and what's the correct approach?**
EKS only supports upgrading one minor version at a time, so 1.27 → 1.30 directly isn't supported — the correct approach is a sequence of upgrades: 1.27 → 1.28 → 1.29 → 1.30, upgrading the control plane first at each step, then node groups, then compatible add-on versions (CNI, CoreDNS, kube-proxy) before moving to the next minor version.

**Q5: A team wants every Pod in the cluster to automatically get a sidecar proxy injected without developers having to add it manually to every Deployment. What Kubernetes mechanism makes this possible, and can you name a real-world example that uses it?**
A Mutating Admission Webhook — it intercepts Pod creation requests and modifies the Pod spec on the fly to inject the sidecar container before it's persisted. Istio's automatic Envoy sidecar injection is a real-world example built on exactly this mechanism.

**Q6: Your organization wants to run a fully custom "backup schedule" concept declaratively — `kind: BackupSchedule` with a cron-like spec — and have it actually provision and manage backup Jobs automatically. What two Kubernetes extensibility mechanisms together make this possible?**
A CustomResourceDefinition to define the new `BackupSchedule` API type/schema, and an Operator (custom controller) that watches for `BackupSchedule` objects and implements the actual logic to create/manage the underlying Jobs/CronJobs and reconcile them over time — the CRD alone is just a schema; the Operator is what makes it functionally meaningful.

**Q7: A Pod using the AWS VPC CNI fails to schedule on an otherwise resource-available node, and `kubectl describe pod` mentions something about insufficient IP addresses. What's going on?**
Because the AWS VPC CNI assigns every Pod a real VPC IP address via ENIs, a node's Pod capacity can be constrained by the number of IP addresses its attached ENIs can support for its instance type — not just CPU/memory. Fixes include using a larger instance type (more ENI capacity), enabling prefix delegation mode in the CNI for higher IP density, or adjusting subnet sizing.

---

## Core Interview Q&A

**Q: What's the first command you run when a Pod isn't behaving as expected, and why?**
A: `kubectl describe pod <pod>` — its Events section almost always states the specific reason (scheduling failure, image pull error, probe failure, OOM kill), making it the fastest path to root cause before diving into logs.

**Q: A container was OOMKilled. What exit code do you expect to see, and what does it actually mean?**
A: Exit code 137 — the container exceeded its configured memory limit and the kernel's OOM killer terminated it. This is about the container's own limit, not necessarily overall node memory pressure.

**Q: Why might a Service have zero entries in its Endpoints, even though matching Pods exist and are Running?**
A: Most commonly a label selector mismatch between the Service and the Pods, or the Pods are Running but failing their readiness probe — only Ready Pods are added as Service endpoints.

**Q: What's the difference between a managed EKS node group and Fargate?**
A: A managed node group still runs on EC2 instances (AWS handles their lifecycle, but they're real, visible nodes you can inspect); Fargate is serverless — there are no nodes to manage at all, each Pod gets isolated, automatically-sized compute, at the cost of losing DaemonSets and some CNI/CSI capabilities.

**Q: What AWS API call actually happens under the hood when a Pod using IRSA calls an AWS service?**
A: `AssumeRoleWithWebIdentity` via AWS STS — the Pod's projected service account token is exchanged for temporary AWS credentials scoped to the IAM role associated with its ServiceAccount.

**Q: What's the version skew policy between kubelet and the control plane, and why does it matter?**
A: The kubelet can be up to two minor versions behind the control plane's API server version — this is what allows rolling, gradual node upgrades across a large fleet instead of requiring every node to upgrade simultaneously with the control plane.

**Q: What's the relationship between a CRD and an Operator?**
A: A CRD only defines a new custom object type/schema in the Kubernetes API — it does nothing on its own. An Operator is the controller that actually watches instances of that custom resource and implements the reconciliation logic to make it functionally meaningful.

**Q: How does a Mutating Admission Webhook differ from a Validating one?**
A: A mutating webhook can modify an object before it's persisted (e.g., injecting a sidecar); a validating webhook can only accept or reject it, with no modification — mutating webhooks run before validating ones in the admission chain.

**Q: Are Kubernetes namespaces sufficient for hard multi-tenant security isolation?**
A: No — namespaces provide logical/soft isolation (naming scope, RBAC/quota/NetworkPolicy boundaries), but tenants still share the same control plane and node kernel. Genuine hard isolation typically requires separate clusters per tenant.
