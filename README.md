# Interview-Preparation

Personal notes for DevOps / Cloud Engineer interview preparation.

---

## 📁 Linux

| # | File | Covers |
| --- | --- | --- |
| 1 | [Basics_architecture.md](./Linux/Basics_architecture.md) | What Linux is and why it's used; kernel vs shell vs userspace; terminal vs shell; Debian vs RHEL vs Amazon Linux vs Alpine distro differences; the Linux Filesystem Hierarchy Standard (`/etc`, `/var`, `/proc`, `/sys`, `/bin` vs `/sbin`, etc.) |
| 2 | [Basic_commands.md](./Linux/Basic_commands.md) | Navigation (`pwd`, `cd`, `ls`), file operations (`cp`, `mv`, `rm`, `mkdir`, `touch`), wildcards, absolute vs relative paths, `find` vs `locate`, viewing files (`cat`, `less`, `head`, `tail -f`), plus utility commands — `echo`, `man`, `date`, `sort`, `split`, `shuf`, `diff`, `wc`, `cmp`, `which`, `bc`, and more |
| 3 | [files_links.md](./Linux/files_links.md) | File permissions (`rwx`, numeric vs symbolic `chmod`), `chown`/`chgrp`, special permissions (SUID, SGID, sticky bit), `umask`, ACLs (`setfacl`/`getfacl`), `chattr`/`lsattr`, and inodes/hard links/soft links in depth |
| 4 | [Users_Groups.md](./Linux/Users_Groups.md) | User types and UID ranges, `/etc/passwd` vs `/etc/shadow` vs `/etc/group`, primary vs secondary groups, `useradd`/`usermod`/`userdel`, `su` vs `sudo`, `/etc/sudoers` and `visudo` |
| 5 | [Process_management.md](./Linux/Process_management.md) | `ps`/`top`/`htop`, process states, foreground vs background jobs (`&`, `jobs`, `fg`, `bg`, `nohup`), zombie vs orphan processes, signals (`SIGTERM`/`SIGKILL`/`SIGHUP`), `kill`/`killall`/`pkill`, `nice`/`renice` |
| 6 | [Package_manager.md](./Linux/Package_manager.md) | Debian (`apt`/`dpkg`), RHEL (`yum`/`dnf`/`rpm`), Alpine (`apk`), and compiling software from source (`./configure && make && make install`) |
| 7 | [Disk_management.md](./Linux/Disk_management.md) | `df` vs `du`, `mount`/`umount`, `lsblk`/`fdisk`/`blkid`, filesystem types (ext4, XFS, etc.), `/etc/fstab`, and LVM (Physical/Volume/Logical volumes) |
| 8 | [System_logs_monitoring.md](./Linux/System_logs_monitoring.md) | `/var/log` structure, `journalctl`, `dmesg`, resource monitoring (`uptime`, `free -h`, swap deep-dive, `vmstat`, `iostat`), the full boot sequence (BIOS/UEFI → GRUB → kernel → systemd), and `systemctl`/units/targets |
| 9 | [Networking_basics.md](./Linux/Networking_basics.md) | `ip`/`ss` (modern) vs `ifconfig`/`netstat` (legacy), `ping`/`traceroute`, `curl`/`wget`, `dig`/`nslookup`, `/etc/hosts` & `/etc/resolv.conf`, TCP vs UDP, common ports, and firewalls (`iptables`/`ufw`/`firewalld`) |
| 10 | [SSH_security.md](./Linux/SSH_security.md) | SSH key pairs, `ssh-keygen`/`ssh-copy-id`, `~/.ssh/config`, `known_hosts`, SCP/SFTP/rsync, file permission hardening, SELinux/AppArmor concepts, and server hardening basics (disabling root login, key-only auth) |
| 11 | [Others.md](./Linux/Others.md) | Namespaces & cgroups (the Linux foundations of containers), `vim`/`nano` basics, `sysctl` kernel tuning, time/timezone management (`timedatectl`, NTP/chrony), and debugging tools (`strace`, `tcpdump`) |

---

## 📁 Shell Scripting

| # | File | Covers |
| --- | --- | --- |
| 1 | [Shell_Scripting.md](./Shell_Scripting/Shell_Scripting.md) | Shebang/permissions, variables & quoting, special variables (`$1`, `$@`, `$?`), reading input, conditionals (`if`/`case`, numeric vs string operators), loops (`for`/`while`/`until`), functions, exit codes & `set -euo pipefail`, arrays, and real script examples (disk alert, backup script, user creation, log monitor) |

---
 
## 📁 Git
 
| # | File | Covers |
| --- | --- | --- |
| 1 | [Git.md](./Git/Git.md) | Why version control exists; local vs centralized vs distributed VCS; Git vs GitHub/GitLab/Bitbucket; Git's four core areas (working directory, staging, local repo, remote); Git internals (objects, commits, branches, `HEAD`, detached `HEAD`); setup (`git config`/`init`/`clone`); everyday workflow (`status`/`add`/`commit`/`log`/`diff`/`show`); branching (`checkout`/`switch`); merging (fast-forward vs three-way, merge conflicts); and more advanced topics (rebase, cherry-pick, stash, tags, remotes) |
 
---
 
## 📁 Docker
 
| # | File | Covers |
| --- | --- | --- |
| 1 | [Docker.md](./Docker/Docker.md) | What Docker is, why it's used, and its key features; core architecture (Docker Engine, `dockerd`, containerd, runc, OCI, image layers/overlay2); images & Dockerfile deep-dive (layer caching, `CMD` vs `ENTRYPOINT`, multi-stage builds, `.dockerignore`, base image selection, BuildKit); container runtime behavior (lifecycle, restart policies, resource limits, the PID 1 problem, exec vs attach, logging drivers); networking (bridge drivers, default vs user-defined bridge DNS, port mapping); storage (volumes vs bind mounts vs tmpfs); Docker Compose; security (non-root, Trivy scanning, secrets, read-only filesystems, capabilities); registries & CI/CD integration (ECR, tagging strategy, GitHub Actions); Docker Swarm (conceptual); troubleshooting real-world scenarios; and the Docker → Kubernetes bridge — plus scenario-based questions and full interview Q&A |
 
---
 
## 📁 Kubernetes (K8s)
 
| # | File | Covers |
| --- | --- | --- |
| 1 | [K8s_architecture.md](./K8s/K8s_architecture.md) | What Kubernetes is, why it's used over Docker/Compose/Swarm, real-world uses, and key features; cluster architecture (Control Plane vs Worker Nodes); control plane components (API Server, etcd, Scheduler, Controller Manager, Cloud Controller Manager); node components (kubelet, kube-proxy, containerd); declarative vs imperative management; `kubectl` and the API server; YAML manifest structure |
| 2 | [pods_and_networking.md](./K8s/pods_and_networking.md) | Pods (single vs multi-container, sidecar pattern), ReplicaSet, Deployment (rolling updates/rollback/revisions), StatefulSet, DaemonSet, Job/CronJob, Init Containers, Pod lifecycle phases & container states — each with YAML examples; Kubernetes networking model (flat network, every Pod gets an IP), Services (ClusterIP/NodePort/LoadBalancer/ExternalName), Ingress & Ingress Controllers, kube-proxy (iptables/IPVS), CoreDNS, NetworkPolicies, and CNI plugins (Calico, AWS VPC CNI) |
| 3 | [volumes_and_configs.md](./K8s/volumes_and_configs.md) | Kubernetes Volumes vs Docker volumes, PersistentVolume & PersistentVolumeClaim, StorageClass & dynamic provisioning, access modes (RWO/ROX/RWX), EBS vs EFS CSI drivers (EKS); ConfigMaps and Secrets (definitions, base64 vs real encryption, env vars vs mounted volumes, immutability) — each with YAML examples |
| 4 | [probes_and_scheduling.md](./K8s/probes_and_scheduling.md) | Resource requests vs limits, QoS classes (Guaranteed/Burstable/BestEffort), node selectors & node affinity/anti-affinity, taints & tolerations, Pod affinity/anti-affinity, HPA, VPA (conceptual), Cluster Autoscaler vs Karpenter; liveness/readiness/startup probes, rolling updates & rollback, deployment strategies (Recreate/RollingUpdate/Blue-Green/Canary), Pod Disruption Budgets; RBAC (Role/ClusterRole/RoleBinding/ClusterRoleBinding), ServiceAccounts, Namespaces, Pod Security Standards, NetworkPolicies (security angle), secrets management best practices, and IRSA in depth |
| 5 | [others.md](./K8s/others.md) | Troubleshooting real-world scenarios (Pod `Pending`, `CrashLoopBackOff`, `ImagePullBackOff`, Service not routing traffic, Node `NotReady`, OOMKilled Pods, DNS resolution issues) with diagnostic commands; EKS-specific topics (control plane vs self-managed vs Fargate, IRSA deep dive, AWS Load Balancer Controller, VPC CNI networking & IP exhaustion, EKS upgrade/version skew policy, Cluster Autoscaler vs Karpenter); and advanced extensibility concepts (CRDs, the Operator pattern, Mutating/Validating Admission Webhooks, multi-cluster/multi-tenancy) |
| 6 | [observability.md](./K8s/observability.md) | The built-in day-to-day debugging toolkit — `kubectl logs` (incl. `--previous`/`-f`), `kubectl describe` (Events section as first port of call), and `kubectl get events` (cluster-wide triage, `--field-selector`); Metrics Server and `kubectl top` (and how it differs from a full monitoring stack); and reading liveness/readiness probe failures directly from Pod events as a diagnostic shortcut. *(Prometheus/Grafana intentionally excluded — covered in a separate, dedicated monitoring deep-dive.)* |
 

---
 
## 📁 Helm & Argo CD
 
| # | File | Covers |
| --- | --- | --- |
| 1 | [helm.md](./Helm_and_Argocd/helm.md) | What Helm is and the problem it solves; Helm 2 vs Helm 3 architecture (Tiller removal, release state stored as Secrets); Charts, Values (and precedence order), Templates (built-in objects, control structures, Sprig functions, `include` vs `template`), named templates/`_helpers.tpl`; chart dependencies & global values, library charts; the full chart hook lifecycle (pre/post-install/upgrade/rollback/delete, hook weights & delete policies); `helm test`; Releases and the install/upgrade/rollback workflow (`--atomic`/`--wait`); repositories (traditional & OCI); packaging/publishing/SemVer; Helm plugins (`helm-diff`, `helm-secrets`); security considerations; and Helm vs raw manifests vs Kustomize |
| 2 | [argocd.md](./Helm_and_Argocd/argocd.md) | GitOps principles (declarative, versioned, pulled, continuously reconciled) and GitOps vs traditional push-based CI/CD; Argo CD architecture (API Server, Repo Server, Application Controller, Redis, Dex); the Application CRD; the App of Apps pattern; AppProject; sync policies (manual vs automated, self-heal, prune); sync waves & resource hooks; ApplicationSet; Argo CD's own RBAC; and sync status vs health status/drift detection |