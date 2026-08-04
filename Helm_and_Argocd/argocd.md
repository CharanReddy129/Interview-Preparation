# GitOps & Argo CD

---

## GitOps & Argo CD

### Definition: GitOps Principles
**GitOps** is an operational model for managing infrastructure and application deployments where **Git is the single source of truth** for the desired state of a system, and an automated process continuously ensures the live environment matches what's declared in Git.

**Core principles:**
1. **Declarative** — the entire desired state of the system is expressed declaratively (YAML manifests, Helm charts, Kustomize overlays) — not as a sequence of imperative steps.
2. **Versioned and immutable** — the desired state is stored in Git, giving you a full audit trail, easy rollback (just revert a commit), and immutable history of every change.
3. **Pulled automatically** — software agents (like Argo CD) automatically **pull** the desired state from Git, rather than an external CI system **pushing** changes into the cluster.
4. **Continuously reconciled** — agents continuously compare live state to the declared state in Git and act to correct any drift, the same reconciliation-loop idea that underpins Kubernetes controllers itself, just applied at the whole-application/Git level.

### Why GitOps (vs Traditional Push-Based CI/CD Deployment)
| Aspect | Traditional push-based CI/CD | GitOps (pull-based) |
|---|---|---|
| **Who deploys** | The CI pipeline (e.g., Jenkins/GitHub Actions) has direct cluster credentials and runs `kubectl apply`/`helm upgrade` itself, pushing changes into the cluster | An in-cluster agent (Argo CD) pulls the desired state from Git and applies it — the CI pipeline never touches the cluster directly |
| **Cluster credentials** | Must be exposed to the external CI system (a real security concern — CI systems are common attack targets) | Never leave the cluster; the CI pipeline only needs push access to Git, not to Kubernetes |
| **Source of truth** | Effectively whatever the last pipeline run applied — can drift from Git if someone runs `kubectl edit` manually and no one notices | Git, continuously — any manual/out-of-band change is detected as drift and can be automatically corrected |
| **Auditability** | Deployment history lives in CI logs, potentially truncated/rotated | Full Git commit history — who changed what, when, and why (via commit messages/PRs) |
| **Rollback** | Re-run an old pipeline job, or manually reapply an old manifest | `git revert` a commit — the agent detects the reverted desired state and reconciles automatically |
| **Multi-cluster consistency** | Each pipeline run targets a cluster explicitly; easy to have pipelines diverge over time | Each cluster's agent pulls from Git independently — same Git source can drive many clusters consistently |

**Interview framing:** "The biggest practical win I'd highlight is the credentials model — with push-based CI/CD, your CI system needs standing access to production clusters, which is a real attack surface. With GitOps, that access model is inverted: nothing outside the cluster needs cluster credentials at all."

### Argo CD Architecture
Argo CD is a GitOps continuous delivery tool for Kubernetes, made up of several core components running inside the cluster:

- **API Server** — exposes the gRPC/REST API used by the Argo CD CLI, UI, and webhook receivers; handles authentication/authorization and application management operations.
- **Repository Server** — clones and caches Git repositories, and generates the final Kubernetes manifests from source (rendering Helm charts, Kustomize overlays, or plain YAML/Jsonnet) for the Application Controller to compare against the live cluster.
- **Application Controller** — the core reconciliation engine; continuously watches both the live cluster state and the desired state (from the Repo Server) and performs the actual `sync` operation to reconcile drift. This is Argo CD's equivalent of a Kubernetes controller's reconciliation loop, applied to whole applications.
- **Redis** — used for caching (e.g., manifest generation results) to reduce load on the Repo Server for frequently accessed data.
- **Dex (optional)** — an identity service used for SSO integration (OIDC, LDAP, SAML, GitHub/GitLab auth) into the Argo CD UI/CLI.

### The Application CRD (Core Object)
Everything in Argo CD revolves around the **Application** Custom Resource — it defines what to deploy (source: a Git repo, path, and revision) and where to deploy it (destination: a cluster and namespace), plus the sync policy.

**Example:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-app-manifests.git
    targetRevision: main
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### App of Apps Pattern
The **App of Apps** pattern uses one "parent" Argo CD Application whose own source is a Git repo/path containing **other Argo CD Application manifests** (the "child" apps) — so applying a single parent Application bootstraps and manages an entire fleet of applications declaratively.

**Why it's used:** instead of manually running `argocd app create` for every application (dozens/hundreds in a large org), you commit each child Application's manifest to a repo, point one parent Application at that repo, and Argo CD's own reconciliation loop takes care of creating/updating/removing the child Applications to match what's declared — GitOps managing GitOps, all the way down.

**Example — parent app-of-apps Application:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/argocd-apps.git
    targetRevision: main
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
Where `apps/` in that repo contains individual Application YAML files — one per real application (`apps/frontend.yaml`, `apps/backend.yaml`, `apps/monitoring.yaml`, etc.) — each pointing to its own manifest source.

### AppProject
An **AppProject** groups related Applications together and defines **guardrails** around them — which Git repos they're allowed to source from, which clusters/namespaces they're allowed to deploy into, and which Kubernetes resource kinds they're allowed to manage. This is the primary multi-tenancy and access-control mechanism in Argo CD (separate from Kubernetes RBAC, though it can integrate with it).

**Example:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-payments
  namespace: argocd
spec:
  description: Applications owned by the Payments team
  sourceRepos:
    - https://github.com/my-org/payments-*
  destinations:
    - server: https://kubernetes.default.svc
      namespace: payments-*
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
```

### Sync Policies: Manual vs Automated, Self-Heal, Prune
Argo CD compares the Git-declared desired state to the live cluster state, and a **Sync** is the operation that reconciles any detected difference.

- **Manual sync** — Argo CD detects drift (shows the app as `OutOfSync` in the UI/CLI) but takes no action until a human explicitly triggers `argocd app sync my-app` (or clicks Sync in the UI). Common for production environments where teams want a deliberate approval gate before changes actually apply.
- **Automated sync** — Argo CD automatically syncs whenever it detects the live state differs from Git, without waiting for manual approval. Set via `syncPolicy.automated` in the Application spec.
- **Self-heal** (`selfHeal: true`, only meaningful with automated sync) — if someone manually changes a live resource out-of-band (e.g., `kubectl edit deployment`), Argo CD detects this as drift from Git and **automatically reverts it back** to match Git — enforcing Git as the actual source of truth, not just a suggestion.
- **Prune** (`prune: true`) — if a resource that was previously managed by this Application is **removed from Git**, pruning tells Argo CD to actually delete that resource from the live cluster during sync. Without `prune: true`, Argo CD will leave orphaned resources in the cluster even after they're deleted from Git.

**Interview framing — a favorite interview question:** "What's the difference between `selfHeal` and `prune`?" — selfHeal corrects resources that still exist in Git but were manually changed on the cluster; prune removes resources from the cluster that no longer exist in Git at all. They solve two different kinds of drift and are usually enabled together in mature GitOps setups.

### Sync Waves and Hooks
For applications where resources must be applied in a specific order (e.g., a CRD before the custom resource that uses it, or a database migration Job before the app Deployment that depends on the migrated schema), Argo CD supports:
- **Sync Waves** — an annotation (`argocd.argoproj.io/sync-wave: "1"`) on individual resources that controls ordering; lower-numbered waves are applied first, and Argo CD waits for each wave to be healthy before proceeding to the next.
- **Resource Hooks** — annotations (`argocd.argoproj.io/hook: PreSync`, `Sync`, `PostSync`, `SyncFail`) that let you run specific resources (typically Jobs) at defined points in the sync lifecycle — e.g., a `PreSync` hook Job to run database migrations before the main application rolls out.

**Example:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: my-app-migrator:1.0
      restartPolicy: Never
```

### ApplicationSet
**ApplicationSet** is a controller/CRD that generates multiple Argo CD Applications automatically from a single template, driven by **generators** — solving the "I need the same app deployed across 20 clusters" or "I need one Application per folder in this repo" problem without hand-writing 20 Application manifests.

Common generators: `list` (a static list of values), `cluster` (one Application per registered cluster), `git` (one Application per directory or file match in a repo — useful for the App of Apps pattern at scale).

**Example — one Application per cluster using the `cluster` generator:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-multi-cluster
  namespace: argocd
spec:
  generators:
    - clusters: {}
  template:
    metadata:
      name: '{{name}}-my-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/my-app-manifests.git
        targetRevision: main
        path: k8s/overlays/{{name}}
      destination:
        server: '{{server}}'
        namespace: my-app
```

### RBAC in Argo CD
Separate from Kubernetes RBAC, Argo CD has its **own RBAC layer** governing who can do what within Argo CD itself (view/sync/delete Applications, manage AppProjects, etc.), configured via a policy CSV in the `argocd-rbac-cm` ConfigMap, typically combined with SSO group mappings from Dex. This distinction is a common point of confusion — Kubernetes RBAC controls what Argo CD's own ServiceAccount can do to the cluster, while Argo CD RBAC controls what human users can do within Argo CD's own UI/API.

### Health Checks and Drift Detection
Argo CD tracks two independent statuses for every Application:
- **Sync Status** — `Synced` or `OutOfSync` — does the live state match Git?
- **Health Status** — `Healthy`, `Progressing`, `Degraded`, `Missing`, `Suspended` — is the application actually working correctly at runtime? (e.g., a Deployment can be perfectly `Synced` but still `Degraded` if its Pods are crash-looping).

Argo CD has built-in health check logic for standard Kubernetes resources (Deployments, StatefulSets, Ingress, PVCs, etc.), and supports **custom health checks** (written in Lua) for CRDs/custom resources that Argo CD doesn't understand out of the box.

**Interview framing:** "Sync status and health status answer two different questions — Synced tells you if Git and the cluster agree, Healthy tells you if what's running is actually working. You need both green to be confident a deployment is genuinely fine."

---

## Scenario-Based Questions

**Q1: A developer manually scales a Deployment from 3 to 10 replicas directly with `kubectl scale`, bypassing Git entirely. What happens if the Argo CD Application has `selfHeal: true`?**
Argo CD's Application Controller detects the live replica count no longer matches what's declared in Git, marks the Application `OutOfSync`, and — because `selfHeal` is enabled — automatically re-syncs to revert the replica count back to what Git declares (3), without waiting for a human to notice or intervene.

**Q2: Your team removes a ConfigMap from the Git repo because it's no longer needed, but the ConfigMap is still present in the cluster after the next sync. Why, and how do you fix it?**
`prune: true` isn't enabled on the Application's sync policy — without it, Argo CD only adds/updates resources present in Git but won't delete resources that have been removed from Git. Enabling `prune: true` in `syncPolicy.automated` (or running a manual sync with the prune flag) will clean up the orphaned ConfigMap.

**Q3: You need a database migration Job to run and complete successfully before the new application version's Pods start rolling out during a sync. How do you model this in Argo CD?**
Use a `PreSync` resource hook on the migration Job (`argocd.argoproj.io/hook: PreSync`) — Argo CD will run and wait for it to succeed before proceeding with the rest of the sync (or the app's own sync wave), ensuring the schema is migrated before Pods depending on it start.

**Q4: Leadership asks why moving from Jenkins-driven `kubectl apply` deployments to Argo CD improves your security posture. What's your answer?**
With Jenkins pushing changes, Jenkins needs standing, direct credentials to the production cluster — a real target for compromise. With Argo CD pulling from Git, no external CI/CD system needs cluster credentials at all; Argo CD's in-cluster agent pulls and applies changes itself, and Jenkins' role shrinks to only building/pushing images and updating manifest files in Git (which it already needs access to anyway).

**Q5: You need to deploy the same application, with slightly different config, across 15 different EKS clusters, and want new clusters to automatically get the app without manually creating 15 separate Application objects. What Argo CD feature fits this?**
ApplicationSet with a `cluster` generator — it automatically generates one Application per registered cluster from a single template, so onboarding a new cluster (registering it with Argo CD) automatically produces a new Application for it without any manual YAML authoring.

**Q6: An Application shows `Synced` in the Argo CD UI, but the actual service is returning errors and users are reporting downtime. How is this possible, and what would you check?**
`Synced` only confirms the live cluster state matches Git — it says nothing about whether the running application is actually healthy. Check the Application's **Health Status** (likely `Degraded` or `Progressing`) and drill into the underlying resources (`kubectl describe pod`, probe failures, crash loops) — Sync status and Health status are independent signals.

---

## Core Interview Q&A

**Q: What is GitOps, in one sentence?**
A: An operational model where Git is the single source of truth for a system's desired state, and an automated agent continuously and declaratively reconciles the live environment to match it.

**Q: What's the key architectural difference between GitOps and traditional push-based CI/CD?**
A: In push-based CI/CD, the CI system pushes changes into the cluster using standing cluster credentials. In GitOps, an in-cluster agent (like Argo CD) pulls the desired state from Git and applies it itself — cluster credentials never need to leave the cluster.

**Q: What does the Argo CD Application Controller actually do?**
A: It's the core reconciliation engine — it continuously compares the live cluster state to the desired state (rendered from Git by the Repo Server) and performs sync operations to correct any drift, similar to how a native Kubernetes controller reconciles actual vs desired state.

**Q: Explain the App of Apps pattern.**
A: It's a pattern where a single parent Argo CD Application's source is a Git repo/path containing other Application manifests as its resources — applying that one parent bootstraps and manages an entire fleet of child applications declaratively, rather than manually creating each Application individually.

**Q: What's the difference between selfHeal and prune?**
A: selfHeal automatically reverts resources that still exist in Git but were manually changed on the live cluster back to match Git. Prune deletes resources from the live cluster that have been removed from Git entirely. Both address drift, but different kinds of it.

**Q: What's the difference between an Application's Sync Status and Health Status?**
A: Sync Status indicates whether the live cluster state matches what's declared in Git. Health Status indicates whether the actual running application is functioning correctly (e.g., Pods passing health checks) — an app can be Synced but still Degraded, so both need to be checked.

**Q: What's the purpose of an AppProject?**
A: It groups related Applications and enforces guardrails — restricting which Git repos an Application can source from, which clusters/namespaces it can deploy to, and which resource kinds it's allowed to manage — serving as Argo CD's primary multi-tenancy boundary.

**Q: When would you use ApplicationSet instead of individual Application objects?**
A: When you need to generate many similar Applications programmatically — e.g., one per cluster, one per Git directory, or one per entry in a list — rather than hand-authoring each Application manifest individually.

**Q: How would you sequence a database migration before an app deployment in an Argo CD sync?**
A: Using a PreSync resource hook on the migration Job (or sync waves if ordering multiple resources), so Argo CD runs and waits for the migration to succeed before proceeding with the rest of the sync.
