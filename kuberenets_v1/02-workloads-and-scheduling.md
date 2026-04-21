# 02 — Workloads and scheduling

## 1. Pod

Smallest deployable unit: one or more containers sharing network namespace (one IP per Pod), optional shared volumes.

**Pod lifecycle phases:** `Pending` → `Running` / `Succeeded` / `Failed` → `Unknown` (node unreachable).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
  labels:
    app: demo
spec:
  containers:
    - name: app
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

---

## 2. Probes

| Probe | Purpose |
|-------|---------|
| **livenessProbe** | Restart container if unhealthy |
| **readinessProbe** | Remove Pod from Service endpoints if not ready |
| **startupProbe** | For slow-start apps; disables other probes until success |

```yaml
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:
        tcpSocket:
          port: 8080
        periodSeconds: 3
```

---

## 3. Resources, QoS, and eviction

- **requests:** used for scheduling and guaranteed resources.
- **limits:** hard cap (CPU throttled; OOM kill if memory exceeded).

**QoS classes (from requests/limits):**

| Class | Condition |
|-------|-----------|
| **Guaranteed** | limits = requests for all containers (and all have limits) |
| **Burstable** | requests set, not matching Guaranteed |
| **BestEffort** | no requests/limits |

Under **node pressure**, kubelet evicts **BestEffort** first, then **Burstable**, then **Guaranteed** last.

---

## 4. Deployment and ReplicaSet

- **ReplicaSet:** ensures N replicas of a Pod template.
- **Deployment:** manages ReplicaSets; handles **rolling updates** and **rollbacks**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
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
        - name: nginx
          image: nginx:1.25-alpine
```

**Commands:** `kubectl rollout status deployment/web`, `kubectl rollout undo deployment/web`.

---

## 5. StatefulSet

For **stable network identity** and **ordered** deploy/scale:

- Stable Pod names: `app-0`, `app-1`, …
- Stable DNS: `app-0.app.default.svc.cluster.local`
- Often paired with **volumeClaimTemplates** for per-replica storage.

Use for databases, Kafka, ZooKeeper-style workloads **when** you need identity + ordered operations (many DBs still run better on VMs/operators—interview nuance).

---

## 6. DaemonSet

Exactly one Pod per matching node (or all nodes). Use for log collectors, node agents, CNI components.

---

## 7. Job and CronJob

- **Job:** runs Pod(s) to completion; `completions`, `parallelism`, `backoffLimit`.
- **CronJob:** time-based schedule (`spec.schedule` cron syntax).

---

## 8. initContainers and sidecars

- **initContainers** run to completion **before** main containers start (ordering, migrations).
- **Sidecar:** extra container in same Pod (e.g., envoy, log shipper).  
  *Note:* Native **sidecar containers** (restartPolicy on init) exist in newer versions — check cluster version in interviews.

---

## 9. Scheduling deep dive

The **scheduler** filters nodes (Predicates), then ranks them (Priorities). **Affinity**, **anti-affinity**, **taints/tolerations**, and **topology spread** all influence which node a Pod lands on.

### 9.1 `nodeSelector` (simplest)

Match nodes that have **exact labels**. No “preferred” or “OR” logic—use **node affinity** for that.

```yaml
spec:
  nodeSelector:
    disktype: ssd
    accelerator: nvidia-t4
```

---

### 9.2 Node affinity (node labels only)

Two scheduling phases in the field name:

| Suffix | Meaning |
|--------|---------|
| **…Scheduling…** | Evaluated **when scheduling** the Pod |
| **…Execution…** | What to do **after** scheduling if labels change (`IgnoredDuringExecution` = do not evict/re-schedule automatically) |

| Block | Behavior |
|-------|----------|
| **requiredDuringSchedulingIgnoredDuringExecution** | **Hard** rules — Pod stays `Pending` if no node matches |
| **preferredDuringSchedulingIgnoredDuringExecution** | **Soft** rules — `weight` 1–100, higher = stronger preference |

**Operators:** `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt` (for numeric-ish labels).

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values: ["linux"]
              - key: node-role.kubernetes.io/worker
                operator: Exists
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-east-1a", "us-east-1b"]
        - weight: 20
          preference:
            matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values: ["m5.large", "m5.xlarge"]
```

**Interview tip:** `nodeSelector` is a subset of node affinity; use affinity when you need **OR** across `nodeSelectorTerms` (list of terms = OR), multiple expressions inside one term = **AND**.

---

### 9.3 Pod affinity (co-locate with other Pods)

**podAffinity** schedules **near** Pods that match a `labelSelector` in the same **topology domain**.

- **topologyKey** — the node label that defines “near.” Common values:
  - `kubernetes.io/hostname` — same node
  - `topology.kubernetes.io/zone` — same AZ
  - `topology.kubernetes.io/region` — same region
- **namespaces** / **namespaceSelector** — limit which namespaces the “other” Pods can live in (omit to mean **this Pod’s namespace** only, unless cluster default differs—always verify in docs for your version).

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: redis
          topologyKey: kubernetes.io/hostname
```

**Use case:** Place app Pods on a node that already runs a **local cache** or **log shipper** sidecar fleet you target by labels.

---

### 9.4 Pod anti-affinity (spread away from other Pods)

**podAntiAffinity** pushes replicas **apart** in the topology domain—classic **HA** pattern so one node failure does not take all replicas.

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: web
          topologyKey: topology.kubernetes.io/zone
```

- **preferred** — “try not to pack together” but schedule anyway if impossible.
- **required** — “never violate” — can leave Pods `Pending` if not enough zones/nodes.

**Gotcha:** Anti-affinity is **symmetric** in effect—scheduling Pod A considers existing Pods; very large spreads can make scaling **slow** or block replicas (`Pending`).

---

### 9.5 Pod topology spread constraints

Declarative **maxSkew** across a topology domain (often zones or nodes). Works well with Deployments for **even spread**.

| Field | Meaning |
|-------|---------|
| **maxSkew** | Max difference in Pod count between any two domains |
| **topologyKey** | Node label defining domains (e.g. zone) |
| **whenUnsatisfiable** | `DoNotSchedule` (hard) or `ScheduleAnyway` (soft) |
| **labelSelector** | Which Pods count toward the spread |

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: web
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: web
```

**vs anti-affinity:** Spread constraints express **counts per domain** explicitly; anti-affinity expresses **pairwise** “don’t sit with myself.” You can combine both—know **scheduling latency** and **Pending** risk.

---

### 9.6 Taints (on nodes) and tolerations (on Pods)

**Taint** — property on a **Node** that repels Pods unless they **tolerate** it.

```bash
kubectl taint nodes node1 dedicated=gpu:NoSchedule
kubectl taint nodes node1 node.kubernetes.io/unreachable:NoExecute --overwrite
```

| Effect | Meaning |
|--------|---------|
| **NoSchedule** | Do not place new Pods here (unless tolerated) |
| **PreferNoSchedule** | “Try” to avoid; not guaranteed |
| **NoExecute** | Like NoSchedule **plus** evict running Pods without a matching toleration after a delay |

**Toleration** on Pod — matches taint `key`, `value`, `effect`, `operator` (`Equal` or `Exists`).  
`Exists` with empty key matches **all** taints of that effect (use carefully).

```yaml
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
    - key: "node.kubernetes.io/not-ready"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300
```

**tolerationSeconds** — only for **NoExecute**; after this many seconds without the toleration matching, eviction can occur (for tolerations that “wait out” node problems).

**Order of evaluation (mental model):** Node must satisfy **affinity** rules; node must have **enough resources**; node’s **taints** must be **tolerated** by the Pod; finally **topology spread** and **anti-affinity** narrow or reorder choices.

**DaemonSets:** The controller adds **default tolerations** so system daemons still run on tainted nodes (e.g. `not-ready`, `unreachable`). That is why you still see `fluent-bit` on GPU tainted nodes while normal Deployments do not land there.

---

### 9.7 Other scheduling concepts (quick refs)

| Concept | Notes |
|---------|--------|
| **spec.nodeName** | Bypasses scheduler—**breaks** affinity/taints logic; only for special cases |
| **Pod overhead** | RuntimeClass / cgroup accounting for sandbox overhead |
| **RuntimeClass** | Select alternate runtime (e.g. gVisor, Kata) per Pod |
| **SchedulerName** | Non-default schedulers (e.g. volcano) for batch/ML |
| **Extended resources** | `nvidia.com/gpu`—request like CPU/memory; **device plugins** advertise capacity |
| **Pod priority & preemption** | See §12—high priority can evict lower to fit |

---

## 10. PodDisruptionBudget (PDB)

Limits **voluntary** disruptions (drain, rolling update) so a minimum number of Pods stay available.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

---

## 11. Horizontal Pod Autoscaler (HPA)

Scales Deployment/ReplicaSet/StatefulSet based on **metrics** (CPU/memory via metrics-server, or custom metrics). **VPA** adjusts requests/limits (often as a separate component).

---

## 12. Priority and preemption

**PriorityClass** + Pod `priorityClassName`: higher priority Pods can **preempt** lower priority Pods when resources are scarce.

---

## Quick recap

- Pods hold containers; **Deployments** manage stateless scale; **StatefulSets** give identity + ordered rollout.
- Probes control restarts and **Service** readiness.
- **Node affinity** = schedule on nodes with label rules (hard vs soft). **Pod affinity** = co-locate in a **topologyKey** domain; **pod anti-affinity** = spread apart for HA.
- **Topology spread constraints** balance replica counts across domains (`maxSkew`, `DoNotSchedule` vs `ScheduleAnyway`).
- **Taints** repel Pods; **tolerations** allow scheduling on tainted nodes; **NoExecute** + **tolerationSeconds** control eviction timing.
- **PDB** protects availability during voluntary disruptions; **priority/preemption** can displace lower-priority workloads.

### Interview prompts

1. Difference between liveness and readiness? What happens if you only set liveness?
2. How does a rolling update work under the hood (ReplicaSets)?
3. When would you choose StatefulSet over Deployment?
4. **Node affinity** `requiredDuringScheduling` vs `preferredDuringScheduling`—when does a Pod stay `Pending`?
5. Give an example of **pod anti-affinity** improving HA; what breaks if you make it `required` with too few nodes?
6. How do **topology spread constraints** differ from **pod anti-affinity**?
7. Compare **NoSchedule**, **PreferNoSchedule**, and **NoExecute**; what is **tolerationSeconds** for?
8. How do taints interact with DaemonSets?
