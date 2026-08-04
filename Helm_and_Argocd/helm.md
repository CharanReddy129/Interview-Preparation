# Helm - Complete Notes

---

### Definition: What Helm Is and the Problem It Solves
**Helm** is the package manager for Kubernetes — often described as "apt/yum/npm for Kubernetes". It packages a collection of Kubernetes manifests (Deployments, Services, ConfigMaps, Ingress, etc.) into a single reusable, versioned unit called a **Chart**, which can be templated, configured, installed, upgraded, and rolled back as one cohesive application, rather than managing a pile of loose YAML files by hand.

**The problem Helm solves:**
- **Raw manifests don't scale well for real applications.** A moderately complex app might need 8-15 separate YAML files (Deployment, Service, ConfigMap, Secret, Ingress, HPA, ServiceAccount, PVC...). Applying, updating, and tracking all of them together as one unit with plain `kubectl apply -f` is error-prone and has no built-in versioning or rollback story.
- **No native templating in raw Kubernetes manifests.** You can't natively parameterize a manifest (e.g., "use this replica count in dev, that one in prod") without external tooling — every environment needs its own copy-pasted, hand-edited YAML, which drifts and duplicates easily.
- **No release/versioning concept.** Plain `kubectl apply` has no built-in notion of "this is version 3 of my app's deployment as a whole" or a one-command way to revert an entire application (all its resources together) to a previous known-good state.
- **Sharing and reusing application definitions is hard.** Without Helm, distributing a "install Prometheus" package to others means sharing a folder of YAML and hoping they edit the right values — Helm standardizes this via publicly shareable Charts (Artifact Hub, similar in spirit to Docker Hub for images).

**Interview:** "Helm does for Kubernetes manifests roughly what a package manager does for software installation — parameterized, versioned, shareable, and reversible, instead of a folder of static YAML files you edit by hand for every environment."

---

### Charts
A **Chart** is a Helm package — a directory (or packaged `.tgz` archive) containing everything needed to deploy an application on Kubernetes: templated manifests, default configuration values, and metadata.

**Standard chart directory structure:**
```
my-app/
├── Chart.yaml           # metadata: chart name, version, app version, dependencies
├── Chart.lock            # pinned/resolved dependency versions (like a lockfile)
├── values.yaml           # default configuration values
├── values.schema.json     # optional: JSON Schema to validate values input
├── charts/                # dependency sub-charts (vendored .tgz or unpacked)
├── templates/             # templated Kubernetes manifest files
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── NOTES.txt          # printed to the user after install/upgrade — usage instructions
│   └── _helpers.tpl       # reusable named template snippets/functions
└── .helmignore            # files to exclude when packaging
```

**Example — `Chart.yaml`:**
```yaml
apiVersion: v2
name: my-app
description: A Helm chart for my application
type: application         # or "library" for a library chart
version: 1.2.0             # chart version — must be SemVer, increments with chart changes
appVersion: "2.0.1"         # version of the actual application being deployed (informational)
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

**`NOTES.txt` (worth knowing — a nice touch that shows Helm fluency):**
A special template file whose rendered output is printed to the terminal immediately after `helm install`/`upgrade` completes — commonly used to show users how to access the just-deployed application (e.g., "run `kubectl port-forward` to reach it," or print the generated Ingress URL).

---

### Values
**Values** are the configuration inputs that get injected into a Chart's templates — this is what makes a Chart reusable across environments instead of hardcoded.

**Example — `values.yaml`:**
```yaml
replicaCount: 3
image:
  repository: my-app
  tag: "2.0.1"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
ingress:
  enabled: false
```

**Values Precedence (a very commonly tested detail — highest wins):**
1. `--set` / `--set-string` / `--set-json` flags on the CLI command (highest precedence)
2. `-f`/`--values` file(s) passed on the command line (later files override earlier ones if multiple are passed)
3. The chart's own `values.yaml` (the built-in defaults)

```bash
# Override individual values inline
helm install my-release ./my-app --set replicaCount=5 --set image.tag=2.1.0

# Override using an environment-specific values file
helm install my-release ./my-app -f values-prod.yaml

# Multiple -f files: later ones win on conflicting keys
helm install my-release ./my-app -f values.yaml -f values-prod.yaml -f values-secrets.yaml
```

**`values.schema.json`** — an optional JSON Schema file that Helm validates user-supplied values against before rendering, catching config mistakes (wrong type, missing required field, out-of-range values) early with a clear error instead of a broken/confusing rendered manifest.

---

### Templates — Deep Dive

**Basic value injection:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-my-app
spec:
  replicas: {{ .Values.replicaCount }}
```

**Built-in objects available in every template:**
| Object | What it provides |
|---|---|
| `.Values` | Values from `values.yaml` and CLI overrides |
| `.Release` | `.Release.Name`, `.Release.Namespace`, `.Release.IsInstall`, `.Release.IsUpgrade`, `.Release.Revision` |
| `.Chart` | Metadata from `Chart.yaml` (`.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion`) |
| `.Files` | Access to non-template files in the chart (e.g., reading a static file into a ConfigMap) |
| `.Capabilities` | Info about the target cluster (Kubernetes version, installed API versions) — useful for conditionally rendering manifests compatible with the cluster's actual API version |

**Control structures — conditionals:**
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  rules:
    - host: {{ .Values.ingress.host }}
{{- end }}
```

**Control structures — loops (`range`):**
```yaml
env:
{{- range $key, $value := .Values.extraEnv }}
  - name: {{ $key }}
    value: {{ $value | quote }}
{{- end }}
```

**Control structures — `with` (scoping into a nested object):**
```yaml
{{- with .Values.resources }}
resources:
  requests:
    cpu: {{ .requests.cpu }}
    memory: {{ .requests.memory }}
{{- end }}
```

**Variables:**
```yaml
{{- $fullName := printf "%s-%s" .Release.Name .Chart.Name }}
metadata:
  name: {{ $fullName }}
```

**Common template functions (Sprig library — Helm bundles this, worth naming a few in an interview):**
| Function | Purpose |
|---|---|
| `default "x" .Values.y` | Fallback value if `.Values.y` is empty/unset |
| `quote` | Wraps a value in double quotes (important for YAML string safety) |
| `upper` / `lower` | Case conversion |
| `trunc 63` | Truncate a string (e.g., to satisfy Kubernetes' 63-character name limits) |
| `indent N` / `nindent N` | Indent a block of text by N spaces (`nindent` also adds a leading newline) — essential for correctly embedding multi-line blocks into YAML |
| `toYaml` | Converts a values object/map into YAML — commonly used for passing through arbitrary user-supplied blocks like `resources` or `nodeSelector` unchanged |
| `include` | Calls a named template (defined via `define`) and returns its rendered output as a string, so it can be piped to other functions (e.g., `indent`) |
| `template` | Also calls a named template, but — unlike `include` — its output cannot be piped to further functions; `include` is almost always preferred over `template` for this reason |

**Example combining several of these (a realistic, interview-relevant snippet):**
```yaml
resources:
{{- toYaml .Values.resources | nindent 2 }}
labels:
  {{- include "my-app.labels" . | nindent 4 }}
```

### Named Templates / `_helpers.tpl`
`_helpers.tpl` (the leading underscore tells Helm it's not a manifest to render directly) holds **reusable named template snippets**, defined with `define` and invoked with `include`:

```yaml
{{/* _helpers.tpl */}}
{{- define "my-app.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
{{- end -}}

{{- define "my-app.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```
Used elsewhere in any template as:
```yaml
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
```
**Why this matters:** avoids duplicating the same labels/naming logic across every manifest file in the chart — a single source of truth for common snippets, which is exactly the kind of DRY pattern interviewers look for evidence of when they ask "how would you structure a non-trivial chart."

### Rendering/Debugging Templates
```bash
# Render fully resolved manifests locally, without touching the cluster
helm template my-release ./my-app -f values-prod.yaml

# Render AND validate against the actual target cluster's API (dry-run)
helm install my-release ./my-app --dry-run --debug

# Lint a chart for structural/syntax problems before packaging
helm lint ./my-app
```

---

### Chart Dependencies (Subcharts)
A chart can declare dependencies on other charts (subcharts) directly in `Chart.yaml`:
```yaml
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled       # only install this dependency if this value is true
    tags:
      - caching
```
```bash
helm dependency update ./my-app   # downloads dependencies into charts/ and writes Chart.lock
```

**Global values** — values under a special `global:` key are passed down into every subchart automatically, useful for values that need to be consistent across the parent chart and all its dependencies (e.g., a shared image registry prefix or environment name):
```yaml
global:
  imageRegistry: my-registry.example.com
  environment: production
```
Any subchart can then reference `.Values.global.imageRegistry` regardless of its own local values structure.

**Interview framing:** "Global values are the mechanism for sharing config across a parent chart and all its subcharts without duplicating the same value in five different `values.yaml` files."

### Library Charts
A **library chart** (`type: library` in `Chart.yaml`) contains only reusable template definitions (helper snippets) and **cannot be installed on its own** — it produces no deployable Kubernetes resources by itself. Other charts declare it as a dependency purely to reuse its named templates, avoiding duplicated boilerplate (like standard labels/annotations) across many unrelated charts maintained by the same team/org.

---

### Chart Hooks (Full Lifecycle)
Hooks let specific chart resources (almost always Jobs) execute at defined points in a release's lifecycle, annotated via `helm.sh/hook`:

| Hook | Fires |
|---|---|
| `pre-install` | Before any resources are installed (new release only) |
| `post-install` | After all resources are installed |
| `pre-upgrade` | Before an upgrade is applied to existing resources |
| `post-upgrade` | After an upgrade completes |
| `pre-rollback` | Before a rollback is applied |
| `post-rollback` | After a rollback completes |
| `pre-delete` | Before resources are deleted on `helm uninstall` |
| `post-delete` | After resources are deleted |
| `test` | Runs only via the explicit `helm test` command (see Testing section below) |

**Example:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: my-app-migrator:1.0
      restartPolicy: Never
```
- **`helm.sh/hook-weight`** — controls execution order among multiple hooks of the same type (lower runs first — same idea as Argo CD's sync waves, but Helm's own mechanism, distinct from and unrelated to Argo CD's).
- **`helm.sh/hook-delete-policy`** — controls cleanup of the hook resource: `before-hook-creation` (delete any previous hook resource before running a new one — the default), `hook-succeeded` (delete after it succeeds), `hook-failed` (delete after it fails).

**Interview framing — a common trap:** "Are hooks part of the normal release/rollback tracking?" — No. Hook resources are **not** tracked as part of the release's managed resources in the same way regular templates are, and they're not automatically rolled back by `helm rollback` — this is a real operational gotcha to be aware of (e.g., a `pre-upgrade` migration Job won't automatically "un-migrate" on rollback; that's on you to handle).

---

### Helm Testing
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-test-connection
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: busybox
      command: ['wget']
      args: ['{{ include "my-app.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```
```bash
helm test my-release
```
Runs any resources annotated `helm.sh/hook: test` against a live, already-deployed release — a lightweight smoke test mechanism to verify a release is actually working post-deploy (e.g., "can I reach the service on its expected port").

---

### Releases
A **Release** is a specific, named, deployed instance of a Chart, running with a specific set of Values, in a specific cluster/namespace. The same Chart can be installed multiple times as different Releases, each independently versioned and manageable.

Helm tracks a **revision history** per Release — every `helm upgrade` creates a new revision, enabling rollback to any previous revision (this history is what's stored as those Kubernetes Secrets discussed in the Architecture section above).

```bash
helm list                          # see all releases in the current namespace
helm list -A                       # see all releases across all namespaces
helm history my-release            # see revision history of a release
helm get values my-release          # see the values currently in effect for a release
helm get manifest my-release        # see the actual rendered manifest currently deployed
```

---

### Helm Install / Upgrade / Rollback Workflow
```bash
# Install a chart as a new release
helm install my-release ./my-app -f values-prod.yaml

# Check the status of a release
helm status my-release

# Upgrade an existing release (new chart version and/or new values)
helm upgrade my-release ./my-app -f values-prod.yaml --set image.tag=2.1.0

# Upgrade if it exists, install if it doesn't (common in CI/CD pipelines)
helm upgrade --install my-release ./my-app -f values-prod.yaml

# Automatically roll back if the upgrade fails health checks
helm upgrade my-release ./my-app --atomic --timeout 5m

# Roll back to the previous revision
helm rollback my-release

# Roll back to a specific revision
helm rollback my-release 3

# Uninstall a release (removes all associated Kubernetes resources)
helm uninstall my-release

# Keep release history even after uninstall (for potential rollback/audit)
helm uninstall my-release --keep-history
```

**`--atomic` flag (worth knowing — a strong production-hardening detail):** if the upgrade fails (e.g., new Pods never become healthy within the timeout), Helm automatically rolls the release back to its previous state, rather than leaving it in a half-upgraded, broken condition. Commonly combined with `--wait` (which makes Helm wait for resources to become ready before considering the operation successful in the first place).

**Interview framing:** "`helm upgrade --install` is the pattern I'd use in a CI/CD pipeline — it's idempotent, so the same command works whether it's the first deployment or the hundredth. Adding `--atomic --wait` on top of that gives me automatic rollback if a bad deploy doesn't come up healthy, without needing separate pipeline logic to detect and react to a failed rollout."

---

### Repositories
Helm charts are distributed via **repositories** — either traditional HTTP(S) chart repositories or (in modern Helm) **OCI registries** (the same kind of registry used for container images, e.g., ECR, Docker Hub, GHCR).

```bash
# Add a traditional chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for charts
helm search repo nginx
helm search hub prometheus     # search Artifact Hub

# Install directly from a repo
helm install my-redis bitnami/redis

# OCI-based repositories (modern approach, same registries as container images)
helm registry login my-registry.example.com
helm push my-app-1.2.0.tgz oci://my-registry.example.com/helm-charts
helm install my-release oci://my-registry.example.com/helm-charts/my-app --version 1.2.0
```

**Interview framing:** "OCI support is the direction the ecosystem has moved — it means you can host your own private Helm charts in the exact same registry (like ECR) you're already using for Docker images, instead of standing up and maintaining a separate chart repository."

---

### Packaging and Publishing a Chart
```bash
helm package ./my-app                    # produces my-app-1.2.0.tgz
helm push my-app-1.2.0.tgz oci://my-registry.example.com/helm-charts
```
Chart versions **must follow SemVer 2** (`MAJOR.MINOR.PATCH`) — Helm enforces this and will reject a malformed version. This matters practically because dependency resolution (`dependencies:` in `Chart.yaml`) relies on proper SemVer ranges to work correctly.

---

### Helm Plugins
Helm supports a plugin system to extend its CLI with additional commands. Worth knowing a couple of commonly used ones by name for interview credibility:
- **`helm-diff`** — shows a diff of what an upgrade would actually change before applying it (similar spirit to `kubectl diff`), extremely useful for reviewing changes before a production upgrade.
- **`helm-secrets`** — integrates Helm with SOPS (Secrets OPerationS) to encrypt sensitive values files at rest in Git, decrypting them only at install/upgrade time — a common pattern for keeping secret values GitOps-friendly without committing plaintext secrets.

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade my-release ./my-app -f values-prod.yaml
```

---

### Security Considerations
- **No more Tiller (Helm 3)** — already covered above, but worth repeating as the headline security improvement between versions.
- **Chart provenance/signing** — `helm package --sign` can cryptographically sign a chart, and `helm verify`/`helm install --verify` can confirm a chart hasn't been tampered with before installing it, similar in spirit to Docker Content Trust for images.
- **Secrets in values files** — plain `values.yaml`/`-f` files are not an appropriate place for real secrets (they'd sit in Git in plaintext, and also end up base64-encoded-but-not-encrypted in the release Secret, same caveat as native Kubernetes Secrets). Use `helm-secrets`/SOPS, or better, avoid injecting real secret values through Helm entirely and instead reference an already-existing Kubernetes Secret (created via an External Secrets Operator) from within your templates.
- **RBAC still applies** — since Helm 3 uses your own kubeconfig, whatever RBAC permissions you have (or your CI pipeline's ServiceAccount has) directly bound the blast radius of what a `helm install`/`upgrade` can actually do — there's no separate elevated Helm-level identity to worry about anymore, which is exactly the improvement over Helm 2/Tiller.

---

### Helm vs Raw Manifests vs Kustomize
| Approach | How it works | Strengths | Weaknesses |
|---|---|---|---|
| **Raw manifests** | Plain YAML files applied directly with `kubectl apply -f` | Simplest, no extra tooling, fully transparent | No templating, no versioning/rollback as a unit, painful to manage across many environments, lots of copy-pasted near-duplicate files |
| **Helm** | Templated Charts with Values, installed/upgraded/rolled back as versioned Releases | Full templating language, release/revision tracking with built-in rollback, huge ecosystem of pre-built charts (Artifact Hub), hooks for lifecycle events, dependency management | Go templating syntax has a learning curve and can get hard to read/debug at scale; adds a layer of abstraction and tooling dependency |
| **Kustomize** | Patch-based approach — a `base` set of plain manifests, with `overlays` per environment that patch/override specific fields, no templating language at all | Built into `kubectl` natively (`kubectl apply -k`), works with plain YAML (no new syntax to learn), simple mental model of base + patches | No packaging/versioning/release concept like Helm, no built-in rollback mechanism, less suited for distributing reusable third-party packages |

**When to use which (a common interview question):**
- **Raw manifests** — fine for small, simple, single-environment setups or quick learning/testing.
- **Kustomize** — good fit when you want to manage environment-specific differences (dev/staging/prod) on top of a shared base, without introducing a templating language, and you're comfortable managing rollout/versioning yourself (often paired with GitOps tools like Argo CD, which natively understands Kustomize overlays).
- **Helm** — the standard choice when you need real packaging, versioning, one-command rollback, dependency management, or you're installing third-party software (most popular Kubernetes tools — Prometheus, NGINX Ingress Controller, cert-manager — are distributed as Helm charts, so you'll use Helm regardless of your own team's preference just to consume them).

**Important nuance:** these aren't mutually exclusive — many real-world setups use Helm to install third-party charts, while using Kustomize (or their own Helm charts) for their own applications, and it's common for Argo CD to render either Helm charts or Kustomize overlays as part of a GitOps pipeline.

---

## Scenario-Based Questions

**Q1: You need to deploy the same application to dev, staging, and prod, with different replica counts and resource limits in each, without maintaining three separate copies of your manifests. How does Helm solve this?**
Define one Chart with templated manifests referencing `.Values`, then maintain separate `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml` files with the environment-specific overrides. Each environment is deployed with `helm install`/`upgrade -f values-<env>.yaml` against the same underlying chart — one source of templated truth, environment-specific configuration layered on top.

**Q2: A `helm upgrade` goes out with a broken configuration and the application starts failing in production. What's your immediate recovery step, and how could you have prevented needing it?**
Immediate recovery: `helm rollback my-release` (or a specific revision via `helm rollback my-release <N>`) — reverts the entire release back to a prior known-good revision in one command. Prevention: running the upgrade with `--atomic --wait` in the first place, so Helm automatically detects the failed rollout and rolls back itself without a human needing to react under pressure.

**Q3: You want to see exactly what Kubernetes manifests a Helm chart will produce with your specific values, before actually applying anything to the cluster — and also want to know exactly what would change compared to what's currently deployed. What commands do you use for each?**
`helm template my-release ./my-app -f values-prod.yaml` renders the fully resolved manifests without touching the cluster. For an actual diff against what's live, the `helm-diff` plugin (`helm diff upgrade my-release ./my-app -f values-prod.yaml`) shows exactly what fields would change — `helm template` alone doesn't compare against current state.

**Q4: Your chart needs a database migration to run before new application Pods roll out during an upgrade, and you want the migration Job cleaned up automatically once it succeeds. How do you model this?**
A Job resource annotated with `helm.sh/hook: pre-upgrade` and `helm.sh/hook-delete-policy: hook-succeeded` — Helm runs it before the upgrade proceeds, and deletes it automatically once it completes successfully.

**Q5: A teammate assumes that rolling back a release with `helm rollback` will also undo a database migration that ran as a `pre-upgrade` hook during the original upgrade. Is this correct?**
No — hook resources are not tracked as part of the release's regular managed resources and are not automatically reverted by `helm rollback`. If a migration needs to be reversible, that has to be handled explicitly (e.g., a corresponding down-migration script), since Helm's rollback only reverts the templated release resources themselves, not hook side effects.

**Q6: You need to store a private, internal Helm chart somewhere your CI/CD pipeline can pull it from, and you'd prefer not to stand up a separate chart repository server. What's the modern approach?**
Push the packaged chart to an OCI-compliant registry (e.g., the same ECR repository already used for container images) via `helm push my-app-1.2.0.tgz oci://<registry>/helm-charts`, and install/reference it via `oci://` URLs — avoiding the need for a dedicated traditional HTTP chart repository.

**Q7: Why was Tiller removed in Helm 3, and what changed operationally as a result?**
Tiller was a server-side component with broad, often near-cluster-admin permissions shared across all users of a cluster, regardless of their own individual permissions — a significant security weakness with no per-user granularity. Helm 3 removed it entirely; the `helm` CLI now talks directly to the Kubernetes API server using the caller's own kubeconfig/RBAC permissions, so Helm's effective permissions are exactly whatever the invoking user or CI ServiceAccount already has — no separate elevated identity to secure or reason about.

---

## Core Interview Q&A

**Q: What is Helm, in one sentence?**
A: Helm is the package manager for Kubernetes — it packages a set of Kubernetes manifests into a templated, versioned, installable unit called a Chart, and manages deployed instances of that chart as trackable Releases.

**Q: What's the difference between a Chart, a Release, and Values?**
A: A Chart is the reusable package definition (templates + metadata + default config). A Release is a specific named, deployed instance of a Chart with a specific set of Values, running in a cluster. Values are the configuration inputs that get injected into the Chart's templates to produce the final manifests for that Release.

**Q: What changed between Helm 2 and Helm 3, and why does it matter?**
A: Helm 2 used a server-side component called Tiller with broad, shared cluster permissions — a real security weakness. Helm 3 removed Tiller entirely; the CLI talks directly to the API server using the caller's own RBAC permissions, and release state is stored as Secrets in the release's own namespace instead of centrally in Tiller.

**Q: Where does Helm actually store a release's history/state?**
A: As Kubernetes Secrets in the same namespace as the release (Helm 3), one per revision — this is what `helm history` and `helm rollback` read from and act on.

**Q: What's the difference between `include` and `template` when calling a named template?**
A: Both call a named template defined via `define`, but `include`'s output can be piped into further template functions (like `nindent`), while `template`'s output cannot — which is why `include` is almost always preferred in practice.

**Q: How does Helm resolve which value wins when the same key is set in multiple places?**
A: `--set` flags win over `-f`/`--values` files, which win over the chart's own default `values.yaml`. Among multiple `-f` files, later files on the command line override earlier ones.

**Q: What are Helm hooks, and are they part of the tracked release?**
A: Hooks are resources (usually Jobs) annotated to run at specific lifecycle points (pre-install, post-upgrade, etc.). They are not tracked as part of the release's normal managed resources and are not automatically reverted by `helm rollback` — a common operational gotcha.

**Q: What does `--atomic` do on `helm upgrade`?**
A: It automatically rolls the release back to its previous state if the upgrade fails (e.g., new Pods don't become healthy within the timeout), rather than leaving the release in a broken, half-upgraded state.

**Q: What's a library chart?**
A: A chart of `type: library` that contains only reusable named template definitions and cannot be installed on its own — other charts depend on it purely to reuse shared templating logic (e.g., standard labels) without duplicating it across multiple charts.

**Q: How would you keep real secrets out of your Helm values files while still using Helm to deploy an app that needs them?**
A: Either encrypt values files at rest with a tool like `helm-secrets`/SOPS (decrypted only at install time), or better, avoid passing secret values through Helm entirely — reference an existing Kubernetes Secret (populated via an External Secrets Operator from AWS Secrets Manager/Vault) directly in the chart's templates instead.

**Q: What's the difference between `helm template` and `helm install --dry-run`?**
A: `helm template` renders manifests purely locally without any cluster interaction. `helm install --dry-run` also renders the manifests, but does so by actually connecting to the target cluster and validating against its live API (catching things like invalid API versions for that specific cluster) — `--debug` alongside it shows the computed values too.
