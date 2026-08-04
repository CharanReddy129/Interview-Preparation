# Kubernetes Notes — Part 3
---

## Storage

### Definition: Volumes in Kubernetes (Different from Docker Volumes)
A **Kubernetes Volume** is a directory (potentially backed by various storage media) that is accessible to containers in a Pod and can outlive individual container restarts within that Pod. This is a broader, more abstract concept than a Docker volume.

**Key difference from Docker:**
- A **Docker volume** is host-scoped — tied to a single Docker host's `/var/lib/docker/volumes`, and only useful for persistence on that one machine.
- A **Kubernetes Volume** is Pod-scoped by default (shares the Pod's lifecycle — if the Pod is deleted, most basic volume types like `emptyDir` are deleted too) but Kubernetes provides an abstraction layer (**PersistentVolume**) on top that can be backed by network/cloud storage (AWS EBS, EFS, etc.), decoupling storage lifecycle from any single node or even any single Pod — critical in a cluster where Pods can be rescheduled to any node at any time.

Kubernetes supports many volume types: `emptyDir` (temporary, Pod-lifetime only, useful for scratch space or sidecar-to-app sharing), `hostPath` (mounts a path from the node's filesystem — risky, ties a Pod to a specific node), `configMap`/`secret` (inject configuration as files), and `persistentVolumeClaim` (the standard way to request durable, network-backed storage).

### PersistentVolume (PV) and PersistentVolumeClaim (PVC)
This is a **two-sided abstraction** that decouples "how storage is provisioned" from "how an application requests storage":

- **PersistentVolume (PV)** — a piece of storage in the cluster, provisioned either manually by an administrator or dynamically via a StorageClass. It's a cluster-level resource, independent of any particular Pod, with its own lifecycle.
- **PersistentVolumeClaim (PVC)** — a request for storage made by a user/application, specifying size and access mode. Kubernetes binds the PVC to a matching available PV (or dynamically provisions one via a StorageClass).

**Analogy:** PV is like a physical disk sitting in a storage room; PVC is like a request form saying "I need 10Gi of storage" — Kubernetes matches the request to an available disk (or orders a new one via dynamic provisioning).

**Example — PersistentVolume:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    volumeID: vol-0abcd1234efgh5678
    fsType: ext4
```

**Example — PersistentVolumeClaim:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**Example — mounting a PVC in a Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: app-storage
          mountPath: /data
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: pvc-example
```

### StorageClass and Dynamic Provisioning
Manually pre-creating PVs for every possible storage need doesn't scale. A **StorageClass** defines "a class of storage" (e.g., "fast SSD," "standard HDD") along with the provisioner and parameters needed to create storage **on demand**. When a PVC references a StorageClass (or uses the cluster's default one), Kubernetes automatically provisions a matching PV behind the scenes — this is **dynamic provisioning**, and it's the standard approach in modern clusters (including EKS).

**Example:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: fast-ssd   # triggers dynamic provisioning
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

**Interview framing:** "In practice, you almost never hand-create PVs in a cloud-managed cluster like EKS — you define a StorageClass once, and every PVC referencing it gets its own EBS volume provisioned automatically and on demand."

### Access Modes
Access modes describe how a volume can be mounted, from the perspective of nodes (not individual Pods):

| Mode | Meaning |
|---|---|
| `ReadWriteOnce (RWO)` | Volume can be mounted as read-write by a single node at a time (most common — typical for EBS-backed volumes) |
| `ReadOnlyMany (ROX)` | Volume can be mounted read-only by many nodes simultaneously |
| `ReadWriteMany (RWX)` | Volume can be mounted read-write by many nodes simultaneously (requires a storage backend that supports concurrent multi-node writes, like EFS or NFS — EBS does NOT support this) |

**Common interview** "Can multiple Pods on different nodes write to the same EBS volume at once?" — No, EBS only supports RWO; for that use case you'd need EFS (which supports RWX) instead.

### ConfigMap and Secret as Volume Mounts vs Env Vars
Both ConfigMaps and Secrets can be consumed by a Pod in two different ways:
- **As environment variables** — injected directly into the container's environment at startup. Simple, but the app must restart to pick up any change, and values can leak into process listings/logs more easily.
- **As mounted volumes** — exposed as files inside the container's filesystem (one file per key). This allows some tools (like a config-reloader sidecar) to detect file changes and reload configuration without restarting the whole Pod, and keeps secrets out of environment variable dumps.

### EBS/EFS CSI Drivers (AWS-Specific — EKS)
Kubernetes storage backends are implemented via the **Container Storage Interface (CSI)** — a standard plugin interface, similar in spirit to CNI for networking. On EKS, the two most relevant drivers are:
- **Amazon EBS CSI Driver** — provisions and attaches EBS volumes as PersistentVolumes. Supports `ReadWriteOnce` only (an EBS volume can only attach to one EC2 instance/node at a time), making it the default choice for single-Pod-writer workloads like databases.
- **Amazon EFS CSI Driver** — provisions access to EFS (a managed NFS-based file system). Supports `ReadWriteMany`, making it the right choice when multiple Pods across multiple nodes need concurrent read-write access to the same files (e.g., shared upload directories, shared config across replicas).

Both must be installed as add-ons on an EKS cluster (they're not built in by default) and require appropriate IAM permissions (commonly via IRSA — IAM Roles for Service Accounts) to interact with the underlying AWS storage APIs.

---

## Configuration & Secrets

### ConfigMap
A **ConfigMap** is a Kubernetes object used to store **non-sensitive configuration data** as key-value pairs, decoupling configuration from application/container images so the same image can be reused across environments (dev/staging/prod) with different configuration.

**Use cases:** application config files, environment-specific settings (API endpoints, feature flags), command-line arguments for a container.

**Example — creating a ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

**Example — consuming it as environment variables:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-configmap-env
spec:
  containers:
    - name: app
      image: my-app:1.0
      envFrom:
        - configMapRef:
            name: app-config
```

**Example — consuming it as a mounted volume:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-configmap-volume
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

### Secret
A **Secret** is a Kubernetes object similar to a ConfigMap, but intended for **sensitive data** — passwords, API tokens, TLS certificates, database credentials.

**Base64 encoding vs real encryption — a critical distinction:**
Secret values are stored **base64-encoded**, not encrypted, by default. Base64 is a reversible encoding, not an encryption algorithm — anyone with `kubectl get secret -o yaml` access (or read access to etcd directly) can trivially decode it. This is a very common interview trap: candidates often assume "Secret" implies real security, but on its own it mainly just separates sensitive data from application code/manifests.

**When it's not enough:**
- etcd itself should have **encryption at rest** enabled (via `EncryptionConfiguration`) so Secret data isn't stored in plaintext (base64) on disk within etcd.
- RBAC must tightly restrict who/what can `get`/`list` Secrets.
- For stronger security, integrate with an external secrets manager (AWS Secrets Manager, HashiCorp Vault) via tools like the **External Secrets Operator**, which sync real secrets into the cluster at runtime rather than storing them natively as base64 in etcd — this ties directly to the IRSA-based AWS Secrets Manager integration you'd use on EKS.

**Example — creating a Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=        # base64 for "admin"
  password: cGFzc3dvcmQxMjM=  # base64 for "password123"
```

**Example — consuming it as environment variables:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secret-env
spec:
  containers:
    - name: app
      image: my-app:1.0
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

**Example — consuming it as a mounted volume:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secret-volume
spec:
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
```

### Environment Variable Injection vs Mounted Config — When to Use Which
| Approach | Pros | Cons |
|---|---|---|
| **Env vars** | Simple, widely supported by every app/language, easy to reference | Requires Pod restart to pick up changes; visible in `kubectl describe pod` and process environment (slightly higher exposure risk for secrets); awkward for multi-line config files |
| **Mounted volume** | Can be updated without restarting the Pod (app must watch the file, though kubelet updates mounted ConfigMap/Secret files automatically within a sync period); better for structured config files (YAML/JSON); slightly better secret hygiene (not in env dumps) | Slightly more setup; app must read from filesystem instead of just env, which not all apps support natively |

**Interview framing:** "I use env vars for simple flags/values, and mounted volumes for full config files or anything where I want the option of a live-reload pattern without a full Pod restart."

### Immutable ConfigMaps/Secrets
Setting `immutable: true` on a ConfigMap or Secret prevents any further updates to its data after creation — attempting to modify it will be rejected by the API server.

**Why this matters:**
- **Performance at scale** — the kubelet doesn't need to keep watching an immutable object for changes, reducing load on the API server in large clusters with many ConfigMaps/Secrets.
- **Safety** — prevents accidental changes to configuration that's in active use; if you need to change values, you create a new ConfigMap/Secret (typically with a versioned name) and update the Pod spec/Deployment to reference the new one — which also plays well with rolling updates, since changing the referenced object name triggers a proper rollout.

**Example:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
data:
  APP_ENV: "production"
immutable: true
```

---

## Scenario-Based Questions

**Q1: A Pod using an EBS-backed PVC needs to be accessed read-write by 5 replicas running on different nodes simultaneously. What breaks, and how do you fix it?**
EBS only supports `ReadWriteOnce` — it can only be attached read-write to one node at a time, so additional Pods on other nodes attempting to mount the same PVC will fail to schedule/attach. Fix: switch to an EFS-backed PVC (via the EFS CSI driver), which supports `ReadWriteMany` and allows genuinely concurrent multi-node access.

**Q2: You update a ConfigMap's data, but the application's Pods don't reflect the new values. What's the most likely explanation, and what would you check first?**
If the ConfigMap is consumed as an **environment variable**, updates never propagate without a Pod restart — env vars are only read once at container start. If consumed as a **mounted volume**, the kubelet does update the file on the mounted filesystem within a sync period, but the application itself has to be watching the file/reloading its config — the update reaching the filesystem doesn't guarantee the running process picks it up.

**Q3: A security review flags that your Secrets are "not actually secure." Why might this be a fair criticism, and what would you improve?**
Because Secret values are only base64-encoded by default, not encrypted — this is easily reversible by anyone with API/etcd read access. Improvements: enable etcd encryption at rest, tighten RBAC so only necessary ServiceAccounts/users can read Secrets, and consider migrating sensitive values to an external secrets manager (AWS Secrets Manager/Vault) synced in via an operator rather than stored natively.

**Q4: You need to change a database's connection string used by an already-running Deployment without causing a rollout, ideally with zero downtime and no accidental drift. What's the cleanest pattern?**
Create a new versioned ConfigMap/Secret (e.g., `db-config-v2`), mark it immutable, and update the Deployment's Pod template to reference the new object name — this naturally triggers a proper rolling update (since the Pod template changed) rather than mutating live configuration in place, which is safer and keeps a clear audit trail.

**Q5: A developer asks why they can't just bake the database password into the Docker image via `ENV` in the Dockerfile, since "it's simpler." How do you explain the risk, and what's the Kubernetes-native alternative?**
Baking secrets into the image means they persist in image layer history, are visible via `docker history`/`docker inspect`, and get replicated everywhere that image is pulled — with no ability to rotate the secret without rebuilding and redistributing the image. The Kubernetes-native alternative is a Secret object (ideally backed by an external secrets manager), injected at runtime via env var or mounted volume, decoupled entirely from the image.

---

## Core Interview Q&A

**Q: How is a Kubernetes Volume different from a Docker volume?**
A: A Docker volume is scoped to a single host. A Kubernetes Volume is scoped to a Pod by default, but the PersistentVolume abstraction decouples storage from any specific node or Pod, allowing network/cloud-backed storage (like EBS/EFS) to follow a Pod even if it's rescheduled elsewhere in the cluster.

**Q: What's the difference between a PersistentVolume and a PersistentVolumeClaim?**
A: A PersistentVolume is the actual piece of provisioned storage in the cluster. A PersistentVolumeClaim is a request for storage made by an application, which Kubernetes binds to a matching (or dynamically provisioned) PersistentVolume.

**Q: What does a StorageClass do?**
A: It defines a "class" of storage along with the provisioner and parameters needed to dynamically create a PersistentVolume on demand whenever a PVC references it, instead of requiring an administrator to manually pre-create PVs.

**Q: Explain the difference between RWO, ROX, and RWX access modes.**
A: RWO (ReadWriteOnce) allows read-write mounting by a single node at a time. ROX (ReadOnlyMany) allows read-only mounting by multiple nodes. RWX (ReadWriteMany) allows read-write mounting by multiple nodes simultaneously — EBS only supports RWO, while EFS supports RWX.

**Q: Are Kubernetes Secrets encrypted?**
A: Not by default — Secret data is only base64-encoded, which is trivially reversible, not encrypted. Real protection requires enabling etcd encryption at rest, strict RBAC, and often integration with an external secrets manager.

**Q: When would you use env vars vs a mounted volume for configuration?**
A: Env vars are simplest for individual flags/values but require a Pod restart to pick up changes. Mounted volumes are better for full config files and can be updated on disk without a restart (though the app must detect and reload the file itself).

**Q: Why would you mark a ConfigMap or Secret as immutable?**
A: To reduce API server/kubelet load in large clusters (no need to watch for changes) and to prevent accidental in-place modification — instead you create a new versioned object and update the Deployment to reference it, which also triggers a clean rolling update.

**Q: What's the difference between the EBS CSI driver and the EFS CSI driver on EKS?**
A: The EBS CSI driver provisions block storage that only supports ReadWriteOnce (attached to one node at a time) — good for single-writer workloads like databases. The EFS CSI driver provisions a managed NFS file system supporting ReadWriteMany — needed when multiple Pods across multiple nodes must read/write the same files concurrently.
