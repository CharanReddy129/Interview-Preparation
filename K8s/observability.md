# Kubernetes Notes - Part 6

---

## Observability


### `kubectl logs`, `kubectl describe`, `kubectl get events` — The Day-to-Day Debugging Toolkit
These three commands are the first line of defense for almost every Kubernetes issue, and interviewers frequently probe whether you reach for them in the right order and know what each one actually surfaces.

**`kubectl logs`** — retrieves the stdout/stderr output of a container, i.e., application-level logs.
```bash
kubectl logs <pod>                          # current container's logs
kubectl logs <pod> -c <container>            # specific container in a multi-container Pod
kubectl logs <pod> --previous                # logs from the PREVIOUS (crashed) instance — critical for CrashLoopBackOff
kubectl logs <pod> -f                        # follow/stream logs live, like `tail -f`
kubectl logs <pod> --since=10m                # only logs from the last 10 minutes
kubectl logs -l app=my-app --all-containers  # logs across all Pods matching a label, useful for a Deployment with multiple replicas
```
**Limitation to know:** `kubectl logs` only shows what the container itself wrote to stdout/stderr — it says nothing about *why* the container isn't running, was never scheduled, or is stuck pulling an image. For those, you need `describe`.

**`kubectl describe`** — shows detailed, human-readable information about any object: its full spec, current status/conditions, and — critically — its **Events** section, which is usually the single fastest path to root cause for anything that isn't a pure application logic bug.
```bash
kubectl describe pod <pod>          # scheduling issues, image pull errors, probe failures, OOM kills, restart counts
kubectl describe node <node>         # node conditions (MemoryPressure/DiskPressure/Ready), allocatable resources
kubectl describe deployment <name>   # rollout status, replica counts, associated events
kubectl describe service <name>      # selector, endpoints, port mappings
```
**Interview framing:** "`describe` is almost always my first command on any object that isn't behaving as expected, before I even look at logs — the Events section at the bottom names the actual reason in plain language far more often than people expect."

**`kubectl get events`** — shows a cluster-wide (or namespace-scoped) stream of events, which is what `describe` is actually pulling its Events section from, but without needing to already know which specific object to inspect.
```bash
kubectl get events                                     # events in the current namespace
kubectl get events --sort-by='.lastTimestamp'           # chronological order — usually more useful than the default
kubectl get events -A                                    # across all namespaces
kubectl get events --field-selector involvedObject.name=<pod>   # filter to a specific object
kubectl get events --field-selector type=Warning          # only warnings/errors, filtering out routine noise
```
**Why this matters as a distinct tool from `describe`:** `get events` is useful when you don't yet know *which* object is the problem — e.g., "something in this namespace is failing, what is it?" — where `describe` requires you to already have a specific object name in hand.

**Putting the three together — a realistic debugging sequence:**
```bash
kubectl get pods -o wide                                 # 1. quick overview — what's not Running/Ready?
kubectl describe pod <problem-pod>                        # 2. why? — check Events + container state
kubectl logs <problem-pod> --previous                      # 3. what did the app itself say before it died?
kubectl get events --sort-by='.lastTimestamp' -n <ns>       # 4. broader cluster context if the above wasn't enough
```

### Metrics Server and `kubectl top`
**Metrics Server** is a lightweight, cluster-wide aggregator of resource usage data (CPU/memory) — it collects real-time metrics from the kubelet on every node and exposes them through the Kubernetes **Metrics API**, which several other core features depend on:
- **`kubectl top`** reads directly from this Metrics API.
- The **Horizontal Pod Autoscaler** (from Part 4) requires Metrics Server to be running in order to make CPU/memory-based scaling decisions at all — without it, HPA has no data to act on.

**Important distinction:** Metrics Server only holds **current, in-memory, short-lived** metrics — it is explicitly **not** a monitoring or historical time-series solution. It answers "what's using how much right now," not "what did usage look like over the last week." That's precisely the gap Prometheus (with its persistent time-series storage) fills — Metrics Server and Prometheus are complementary, not competing, and it's a common interview trap to conflate the two.

```bash
kubectl top nodes                       # CPU/memory usage per node
kubectl top pods                         # CPU/memory usage per Pod, current namespace
kubectl top pods -A                       # across all namespaces
kubectl top pods --containers             # broken down per-container within each Pod
kubectl top pods --sort-by=memory          # sort by usage, useful for quickly spotting the heaviest consumers
```

**Interview framing:** "Metrics Server is the lightweight, real-time data source that things like HPA and `kubectl top` depend on — it's not meant to replace Prometheus, it just answers 'right now' questions, while Prometheus is what you'd reach for to actually see trends, set up alerting, or investigate something that happened an hour ago."

**If `kubectl top` returns no data or errors:** the most common cause is that Metrics Server simply isn't installed/running in the cluster (it's not built in by default on all distributions — check `kubectl get deployment metrics-server -n kube-system`), or its Pods themselves are unhealthy.

### Liveness/Readiness Probe Failures as a Debugging Signal
Probes (fully defined in Part 4) aren't just a health-check configuration detail — they're one of the **most common root causes** behind confusing-looking Pod behavior, and knowing how to read probe failures from `describe`/events is a frequently tested practical skill.

**What a liveness probe failure looks like, and what it means:**
```
Warning  Unhealthy  2m (x3 over 4m)  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 500
Normal   Killing    2m               kubelet  Container app failed liveness probe, will be restarted
```
This tells you the container was **restarted specifically because of the probe**, not because the process itself crashed outright — an important distinction when reading `kubectl describe pod`'s restart count and event history, since it points you toward "why is `/healthz` returning 500" rather than "why did my process segfault."

**What a readiness probe failure looks like, and what it means:**
```
Warning  Unhealthy  1m (x5 over 3m)  kubelet  Readiness probe failed: Get "http://10.1.2.3:8080/ready": dial tcp 10.1.2.3:8080: connect: connection refused
```
No `Killing` event follows — because readiness failures don't restart anything, they just pull the Pod out of Service endpoints. This is exactly the "Pod is `Running` but receiving no traffic" scenario from Part 2/Part 4 — if you see readiness failures in events but no restarts, the container process itself is fine; something about its readiness *endpoint* specifically isn't responding correctly (still starting up, dependency not reachable yet, wrong port/path configured).

**Practical diagnostic checklist when you see probe failures in events:**
1. Confirm the probe's `path`/`port` in the manifest actually matches what the app serves (`kubectl describe pod` shows the configured probe).
2. Check whether the failure is a **liveness** (app is genuinely unhealthy/restarting) vs **readiness** (app is fine but not yet ready/reachable) failure — they point to very different problems.
3. Check `initialDelaySeconds`/`periodSeconds`/`timeoutSeconds` — a slow-starting app with too short an `initialDelaySeconds` on its liveness probe can get killed repeatedly before it ever finishes starting (this is exactly what `startupProbe` exists to prevent).
4. `kubectl exec` into the Pod (if still reachable) and manually hit the probe endpoint to see the actual response/error directly.

**Interview framing:** "Probe failures showing up in events are one of my favorite diagnostic shortcuts, because the event log tells you immediately whether you're dealing with a real crash (liveness) or a not-yet-ready dependency (readiness) — that distinction alone usually cuts your investigation time in half."

---

## Scenario-Based Questions

**Q1: A Pod has a restart count of 4, but `kubectl logs` (without `--previous`) shows nothing useful. Walk through how you'd figure out why it keeps restarting.**
Run `kubectl describe pod <pod>` first — the Events section will show whether restarts are coming from a liveness probe failure, an OOM kill, or the container process exiting on its own. Then run `kubectl logs <pod> --previous` to see what the crashed instance actually logged before it died, since the current instance (post-restart) may not have hit the same failure yet.

**Q2: `kubectl top pods` returns "error: Metrics API not available." What does this actually mean, and what would you check?**
It means the Metrics Server either isn't installed in the cluster or its Pods are unhealthy — `kubectl top` (and HPA) depend entirely on it for data. Check `kubectl get deployment metrics-server -n kube-system` and its Pod status/logs; this is unrelated to whether Prometheus is installed, since Metrics Server is a separate, lightweight component Kubernetes' own built-in features rely on.

**Q3: You see repeated readiness probe failures in a Pod's events, but no restarts and no liveness failures. Is the application crashing?**
No — readiness failures alone don't restart anything; they only remove the Pod from Service endpoints. The container process itself is running fine; something about the specific readiness check (endpoint not up yet, wrong path/port, a dependency the readiness check verifies isn't reachable) is failing. This is the classic "Running but receiving no traffic" pattern.

**Q4: A slow-starting Java application keeps getting killed by its liveness probe before it ever finishes initializing. How do you fix this without just disabling the liveness probe?**
Add a `startupProbe` with a generous `failureThreshold`/`periodSeconds` combination that gives the app enough time to fully start — Kubernetes holds off running the liveness (and readiness) probe until the startup probe succeeds, so a legitimately slow-starting app isn't mistaken for an unhealthy one and killed prematurely.

**Q5: You need to quickly find every Pod across the entire cluster with a Warning-level event in the last few minutes, without checking each namespace one by one. What command do you use?**
`kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp'` — filters to warnings only, across all namespaces, in chronological order, giving a fast cluster-wide triage view without needing to already know which object or namespace is affected.

---

## Core Interview Q&A

**Q: What's the difference between `kubectl logs` and `kubectl describe`?**
A: `kubectl logs` shows the application's own stdout/stderr output — what the process itself printed. `kubectl describe` shows Kubernetes-level status and events — scheduling decisions, probe failures, image pull errors, OOM kills — information the application itself never logs because it's about how Kubernetes is managing the container, not what the app is doing internally.

**Q: Why is `--previous` important when using `kubectl logs` on a crashing Pod?**
A: By the time you run the command, the container may have already restarted, so the "current" logs could be empty or unrelated to the crash. `--previous` retrieves logs from the last terminated instance, which usually contains the actual error that caused the crash.

**Q: What does Metrics Server actually provide, and what does it NOT provide?**
A: It provides real-time, current CPU/memory usage per node/Pod via the Kubernetes Metrics API, which `kubectl top` and the Horizontal Pod Autoscaler depend on. It does not provide historical time-series data, dashboards, or alerting — that's the role of a full monitoring stack like Prometheus.

**Q: If HPA isn't scaling a Deployment even though CPU usage is clearly high, what's a foundational thing to check first?**
A: Whether Metrics Server is actually installed and healthy in the cluster — HPA has no usage data to act on at all without it, so this is worth confirming before assuming the HPA configuration itself is wrong.

**Q: How can you tell from `kubectl describe pod`'s events whether a container was killed by a liveness probe versus crashing on its own?**
A: A liveness-probe-triggered restart shows explicit `Unhealthy` events followed by a `Killing` event referencing the probe failure. A self-inflicted crash typically shows the container's own exit code/reason (e.g., `Error`, non-zero exit) without a preceding probe failure event.

**Q: Why doesn't a readiness probe failure show up as a restart?**
A: Because readiness and liveness probes serve different purposes — readiness only controls whether a Pod is included in Service endpoints (routable), while liveness controls whether the container should be restarted. A Pod can fail readiness indefinitely and remain perfectly "alive," just not receiving traffic.
