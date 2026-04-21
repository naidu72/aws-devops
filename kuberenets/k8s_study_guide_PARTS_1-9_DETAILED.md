# Kubernetes study guide — Parts 1–9 (detailed explanations)

**Companion to:** [k8s_study_guide.md](k8s_study_guide.md) (schedule & checklist)  
**Purpose:** One document with **teaching-level explanations** for every Part 1–9. For even deeper dives, use the linked files in `kuberenets/` (see end of each part).

---

## Table of contents

1. [Part 1 — Core Kubernetes resources](#part-1--core-kubernetes-resources)
2. [Part 2 — Storage & volumes](#part-2--storage--volumes)
3. [Part 3 — Networking](#part-3--networking)
4. [Part 4 — Platform architecture (EKS, AKS, bare metal)](#part-4--platform-architecture-eks-aks-bare-metal)
5. [Part 5 — Monitoring & observability](#part-5--monitoring--observability)
6. [Part 6 — Cloud-native & multi-cloud tools](#part-6--cloud-native--multi-cloud-tools)
7. [Part 7 — Hands-on practicals](#part-7--hands-on-practicals)
8. [Part 8 — Mock interviews](#part-8--mock-interviews)
9. [Part 9 — Interview question bank (with answers)](#part-9--interview-question-bank-with-answers)

---

## Part 1 — Core Kubernetes resources

### 1.1 Pod

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers that share network (one IP per pod) and storage volumes. Pods are **ephemeral**: they can be replaced anytime; do not store unique state in a plain Pod without volumes.

**QoS classes** (quality of service, used for eviction ordering):

| Class | Condition | Eviction order |
|-------|-----------|----------------|
| **Guaranteed** | Every container has `requests` = `limits` for CPU and memory | Last evicted |
| **Burstable** | Requests set but not equal to limits (or only some resources set) | Middle |
| **BestEffort** | No requests/limits | First evicted |

**Init containers** run **before** app containers, in order. Use them for: DB migrations, waiting for dependencies, downloading configs, or security setup. They must exit successfully (exit 0) or the Pod never becomes Ready.

**Sidecar pattern:** a second container in the Pod that augments the main app (e.g. Envoy proxy, log shipper). Unlike init containers, sidecars run **with** the main container for the Pod lifetime.

**Probes:**

- **Liveness** — restart container if unhealthy.
- **Readiness** — remove Pod from Service endpoints until ready.
- **Startup** — for slow-starting apps (disables liveness until startup succeeds).

**Deeper:** [core_resources_interview_qa.md](core_resources_interview_qa.md), [core_resources_guide.md](core_resources_guide.md).

---

### 1.2 Deployment

A **Deployment** manages ReplicaSets and provides **declarative rolling updates** for stateless apps. You set `replicas` and a Pod template; Kubernetes keeps that many Pods running.

**Rolling update strategy:**

- `maxSurge` — extra Pods above desired count during update (e.g. 25% or 1).
- `maxUnavailable` — max Pods that can be down during update (e.g. 0 for strict HA).

**Rollback:** `kubectl rollout undo deployment/name` uses ReplicaSet revision history.

**When to use:** Web APIs, workers that do not need stable identity or stable disk per replica.

**Deeper:** same as 1.1; real Helm examples may live in your workspace repos referenced in `k8s_study_guide.md`.

---

### 1.3 StatefulSet vs Deployment

| | Deployment | StatefulSet |
|---|-------------|-------------|
| Pod identity | Random names, interchangeable | Stable name: `app-0`, `app-1` |
| Storage | Usually shared nothing or external | **Per-replica PVC** often via `volumeClaimTemplates` |
| Headless Service | Optional | Often used for stable DNS: `pod-0.app.default.svc.cluster.local` |
| Ordering | No | **Ordered** start/stop/rolling (for clustered DBs, etc.) |

Use **StatefulSet** for: Kafka brokers, etcd, clustered DBs where each member needs identity and its own disk. Use **Deployment** for everything else that is horizontally scalable and stateless.

**Deeper:** [core_resources_guide.md](core_resources_guide.md), [storage_concepts_guide.md](storage_concepts_guide.md).

---

### 1.4 DaemonSet, Job, CronJob

**DaemonSet** ensures **one Pod per matching node** (or all nodes). Typical: node log/metrics agents, CNI helpers, security scanners. Respects taints/tolerations so you can target specific node pools.

**Job** runs a workload **to completion** (batch). Retries and parallelism are configurable. **CronJob** schedules Jobs on a cron schedule.

**Deeper:** [core_resources_interview_qa.md](core_resources_interview_qa.md).

---

### 1.5 Service & Ingress

**Service** types:

- **ClusterIP** (default) — internal VIP; DNS `my-svc.my-ns.svc.cluster.local`.
- **NodePort** — exposes port on every node (30000–32767 range); dev/debug.
- **LoadBalancer** — cloud LB in front (NLB/ALB on AWS); **ExternalName** — CNAME to external host.

**Ingress** (Kubernetes API) — HTTP/HTTPS routing by host/path to Services; needs an **Ingress controller** (e.g. NGINX, AWS LB Controller).

**Deeper:** [networking_deep_dive.md](networking_deep_dive.md).

---

## Part 2 — Storage & volumes

### 2.1 Model: PV, PVC, StorageClass

- **PersistentVolume (PV)** — cluster storage resource (admin-provisioned or dynamic).
- **PersistentVolumeClaim (PVC)** — Pod’s **request** for storage (size, access mode, optionally StorageClass).
- **StorageClass** — template for **dynamic provisioning** (provisioner, parameters, reclaim policy).

**Flow (dynamic):** Pod uses PVC → PVC binds to PV created by provisioner → Pod mounts volume.

**Access modes:** `ReadWriteOnce` (RWO), `ReadOnlyMany` (ROX), `ReadWriteMany` (RWX) — not all storage supports all.

**Reclaim policy:** `Retain`, `Recycle` (deprecated), `Delete` — what happens to PV when PVC is deleted.

**Deeper:** [storage_concepts_guide.md](storage_concepts_guide.md).

---

### 2.2 Volume types (conceptual)

- **emptyDir** — ephemeral, shared between containers in a Pod; lost when Pod dies.
- **hostPath** — node filesystem; security and portability concerns.
- **CSI drivers** — EBS, EFS, cloud disks — production default for stateful data.

**StatefulSet + PVC:** `volumeClaimTemplates` creates one PVC per Pod replica (`data-app-0`, etc.).

---

## Part 3 — Networking

### 3.1 CNI (Container Network Interface)

The CNI plugin gives each Pod an IP and connects Pods across nodes. **Without a CNI, Pods do not get cluster networking.**

Examples:

- **Amazon VPC CNI** — Pods get VPC IPs (ENI/IP limits per instance matter); tight AWS integration.
- **Calico** — strong **NetworkPolicy**, optional BGP; common on-prem.
- **Cilium** — eBPF-based, Hubble observability, advanced policies.

**Deeper:** [networking_deep_dive.md](networking_deep_dive.md) (CNI comparison, IP exhaustion, policies).

---

### 3.2 NetworkPolicy

NetworkPolicy is **default allow-all** if no policies exist; once you apply policies, **only what you allow** is permitted (implementation depends on CNI). You typically combine **default deny** with explicit allows for ingress/egress (including DNS on UDP/TCP 53).

**Deeper:** [networking_deep_dive.md](networking_deep_dive.md).

---

### 3.3 Service discovery & data plane

- **CoreDNS** — cluster DNS; `Service` → ClusterIP; `*.svc.cluster.local`.
- **kube-proxy** — programs rules (iptables, IPVS, or nftables) so ClusterIP reaches backends; modes differ in scale/performance.
- **EndpointSlices** — scalable slices of endpoints behind a Service (replaces large single Endpoint objects).

**Deeper:** [networking_deep_dive.md](networking_deep_dive.md).

---

## Part 4 — Platform architecture (EKS, AKS, bare metal)

### 4.1 Amazon EKS

- **Control plane** — AWS-managed API server, scheduler, controller-manager, etcd; multi-AZ.
- **Data plane** — your nodes: **managed node groups**, self-managed, or **Fargate** (serverless).
- **IRSA (IAM Roles for Service Accounts)** — map Kubernetes ServiceAccount to IAM role via OIDC; **no long-lived AWS keys in Pods**.
- **Add-ons** — VPC CNI, CoreDNS, kube-proxy, EBS CSI driver, etc.

**Deeper:** [eks_architecture_guide.md](eks_architecture_guide.md).

---

### 4.2 Azure AKS

- Azure-managed control plane; **node pools** (system vs user).
- Identity: **managed identity**, workload identity (similar spirit to IRSA).
- Networking: **Azure CNI** (Pods get VNet IPs) vs **Kubenet** (overlay).

**Deeper:** [aks_architecture_guide.md](aks_architecture_guide.md).

---

### 4.3 Bare metal / self-managed

- **kubeadm** bootstraps certificates, static Pods for control plane, joins nodes.
- **HA control plane** — multiple API servers behind LB, **etcd** as odd-numbered quorum (3, 5).
- **Storage / LB** — MetalLB or hardware LB; NFS/Ceph/local volumes.

**Deeper:** [bare_metal_kubernetes_guide.md](bare_metal_kubernetes_guide.md).

---

## Part 5 — Monitoring & observability

### 5.1 Three pillars (what each is for)

1. **Metrics** — Aggregated numbers over time (CPU, memory, request rate, error ratio, saturation). Good for **alerting**, SLOs, and capacity planning because they are cheap to store and query at cluster scale.
2. **Logs** — Discrete events and structured messages. Good for **debugging** “why did this one request fail?” when you have correlation IDs. Cost explodes if you log everything at `DEBUG` in production.
3. **Traces** — End-to-end request path across services (spans, parent/child). Good for **latency breakdown** and finding which hop added delay.

**Interview framing:** Observability means you can **ask new questions** without shipping new code—because you have enough context (labels, exemplars, trace IDs) to pivot from symptom to cause.

### 5.2 Golden signals, USE, RED

- **Golden signals (distributed systems):** Latency, Traffic, Errors, Saturation (sometimes “Saturation” is folded into load).
- **USE method (resources):** Utilization, Saturation, Errors — for nodes, disks, queues.
- **RED method (microservices):** Rate, Errors, Duration — maps cleanly to HTTP/gRPC metrics.

These give you vocabulary for dashboards and alerts that aren’t just “CPU high.”

### 5.3 Prometheus stack (typical on Kubernetes)

- **Prometheus** scrapes HTTP `/metrics` (pull model). Stores time series **locally** (TSDB) or forwards to remote storage (Thanos, Mimir, Cortex).
- **kube-state-metrics** exposes **Kubernetes object state** as metrics (Deployment replicas, Pod phase)—not the same as cAdvisor/node metrics.
- **Node exporter / cAdvisor** (or cloud agent) for host and container runtime metrics.
- **ServiceMonitor** / **PodMonitor** (Prometheus Operator) — declarative scrape configs in Kubernetes API instead of hand-editing Prometheus YAML.

**PromQL patterns:** `rate()` for counters; `histogram_quantile()` for latency from histogram buckets; always watch **cardinality** (too many unique label combinations = memory explosion).

### 5.4 Alerting principles

- Alert on **symptoms** users feel (errors, SLO burn) or **imminent failure** (disk full), not every chart wiggle.
- Use **runbooks** linked from alerts; paging should be **actionable**.
- **“Multi-window, multi-burn-rate”** alerting (Google SRE) reduces noise vs single-threshold flapping.

### 5.5 Logs pipeline on Kubernetes

- **Collection:** Fluent Bit / Fluentd / Vector as **DaemonSet** (per node) is common; sometimes sidecar for strict tenancy.
- **Storage/query:** Loki (label-indexed, pairs well with Prometheus labels), Elasticsearch/OpenSearch (full-text, heavier ops).
- **Correlation:** structured JSON logs with `trace_id`, `span_id`, `request_id` — ties logs to traces.

### 5.6 Tracing and OpenTelemetry

- **OpenTelemetry (OTel)** — vendor-neutral **instrumentation** (SDKs) + **Collector** (receive OTLP, process, export to backends).
- **Jaeger / Tempo / Zipkin** — trace **backends** + UI; modern Jaeger can ingest OTLP.
- **Propagation:** W3C `traceparent` so gateways, meshes, and apps agree on IDs.

**Deeper:** [MONITORING_SCANNING_CLOUD_TOOLS.md](MONITORING_SCANNING_CLOUD_TOOLS.md) (Parts 1–2 + appendix).

---

## Part 6 — Cloud-native & multi-cloud tools

This part is about **secrets, identity, config, and platform services** so workloads stay portable and auditable across clouds.

### 6.1 AWS (patterns you should explain)

- **Secrets Manager** — purpose-built for **secrets** with rotation Lambdas; higher per-secret cost than SSM; good when you need rotation and fine-grained IAM.
- **SSM Parameter Store** — hierarchical **config** and non-rotating secrets; **Standard vs Advanced** parameters; often paired with IAM condition keys.
- **KMS** — envelope encryption: KMS keys wrap data keys; integrates with Secrets Manager, SSM, EBS, S3, **etcd encryption** (EKS).
- **CloudWatch** — metrics, logs, Contributor Insights, alarms; **X-Ray** for AWS-native tracing (compare with OTel elsewhere).
- **IRSA recap** — OIDC provider on EKS + trust policy on IAM role + `serviceAccount.annotations`; **no static keys** in cluster.

### 6.2 Azure (summary)

- **Key Vault** — secrets, keys, certificates; integrate via **CSI Secrets Store driver** or SDKs.
- **Managed identity** — VMs/AKS agents get Azure AD identities; **Workload Identity** federates Kubernetes SAs to Azure AD (conceptual cousin to IRSA).
- **Application Insights / Log Analytics** — APM + centralized logging; **Defender for Cloud** for posture.

### 6.3 GCP (one-liner familiarity)

- **Secret Manager** + **Workload Identity** (GKE) bind Kubernetes SAs to GCP SAs — same “no long-lived keys in Pods” story.

### 6.4 Kubernetes-native secret patterns

- **Secret objects** — base64 encoding is **not encryption**; protect with **RBAC**, **EncryptionConfiguration** for etcd at rest, and narrow `get/list`.
- **External Secrets Operator (ESO)** — reconciles **external** systems of record (AWS, Vault, etc.) into Kubernetes Secrets; fits **GitOps** (only references in Git).
- **Sealed Secrets** / **SOPS** — encrypt Secret manifests **for Git**; different threat model than ESO (no live sync).
- **CSI Secret Store drivers** — mount secrets as volumes from cloud vaults; reduce proliferation of copied Secret objects.

### 6.5 HashiCorp Vault (when interview goes deep)

- **Auth methods:** Kubernetes auth binds ServiceAccount JWTs to Vault policies.
- **Secret engines:** KV v2, dynamic DB creds, PKI issuance.
- **Operational cost:** HA storage backend, unseal, upgrades—often a team-owned platform.

**Deeper:** [MONITORING_SCANNING_CLOUD_TOOLS.md](MONITORING_SCANNING_CLOUD_TOOLS.md) Part 4+.

---

## Part 7 — Hands-on practicals

Hands-on proves you can **translate concepts into working clusters**. Below: **goal**, **how to execute**, **how you know it worked**, **typical failure modes**.

### 7.1 Practical 1 — StatefulSet + persistence

- **Proves:** Stable identity + **per-replica PVC** behavior; data survives Pod restart on same ordinal.
- **Steps:** Deploy StatefulSet with `volumeClaimTemplates`; exec into `pod-0`, write a file under the mounted path; delete the Pod; wait for reschedule; file should still exist on the **same** PVC.
- **Success criteria:** Ordinal hostname resolves; PVC `Bound`; data survives **Pod delete** (not necessarily node loss—that needs snapshot/backup).
- **Fails when:** StorageClass wrong access mode; zone mismatch; PVC stuck `Pending`.

### 7.2 Practical 2 — NetworkPolicy (zero-trust style)

- **Proves:** You understand **default allow** vs explicit deny; DNS egress; label selectors.
- **Steps:** Default-deny ingress/egress → allow namespace-internal → allow egress to `kube-system` CoreDNS (UDP/TCP 53) → allow app→app paths only.
- **Success criteria:** Allowed paths return HTTP 200; denied paths timeout; `nslookup` still works from Pods.
- **Fails when:** CNI doesn’t enforce policies; forgot DNS; used wrong pod labels; policy in wrong namespace.

### 7.3 Practical 3 — Helm troubleshooting

- **Proves:** Template literacy, values precedence, safe upgrades.
- **Steps:** Break something on purpose (wrong image tag); use `helm template` / `helm lint`; fix values; `helm upgrade` with history; optional rollback.
- **Success criteria:** Rendered manifests match intent; upgrade is idempotent; `helm history` shows revision.
- **Fails when:** Editing generated YAML instead of chart; wrong `--namespace`; immutable field change.

### 7.4 Practical 4 — HPA

- **Proves:** **Application** autoscaling vs **node** autoscaling; metrics-server or custom metrics API.
- **Steps:** Install metrics-server; create HPA on Deployment; generate CPU load (`hey`, `wrk`); watch replicas increase; stop load; watch scale down (respect cooldowns).
- **Success criteria:** HPA shows targets; replicas change; no scheduling failures at steady state.
- **Fails when:** No resource requests (HPA needs them for CPU); metrics-server not ready; too aggressive stabilization window.

### 7.5 Practical 5 — Multi-arch image

- **Proves:** Manifest lists; CI builds for **amd64** and **arm64** (Graviton / Apple Silicon clusters).
- **Steps:** `docker buildx build --platform linux/amd64,linux/arm64 --push`; deploy to mixed node pool; verify Pod runs on correct arch.
- **Success criteria:** Image pulls on both architectures; no `exec format error`.
- **Fails when:** Single-arch fat binary; wrong base image; platform not in manifest.

### 7.6 Practical 6 — Backup / restore

- **Proves:** You can separate **etcd/object backup** from **volume snapshots**; restore order.
- **Steps (managed EKS):** Velero backup including namespaces + PV snapshots (CSI); restore to new namespace or cluster drill.
- **Steps (self-managed):** etcd snapshot + restore procedure documented.
- **Success criteria:** Resources reappear; PV data restores where applicable; test app serves traffic.
- **Fails when:** Snapshot permissions missing; backup missing CRDs; restore to wrong region/zone.

### 7.7 Practical 7 — Observability stack

- **Proves:** Metrics + at least one log or trace path **correlated**.
- **Steps:** Prometheus scrapes app metrics; Grafana dashboard; one log query with `trace_id` or `pod` label; optional OTel trace in Jaeger/Tempo.
- **Success criteria:** Dashboards show real traffic; you can pivot from spike in errors to representative logs/traces.
- **Fails when:** ServiceMonitor wrong labels; scrape TLS mismatch; cardinality explosion.

### 7.8 Practical 8 — Security scanning (shift-left)

- **Proves:** Image vulnerabilities caught **before** prod; optional runtime signal.
- **Steps:** Trivy (or similar) in CI on built image; fail build on Criticals per policy; optional Falco rule for unexpected syscall.
- **Success criteria:** Pipeline fails fast; report is actionable (package + fix version).
- **Fails when:** Scanning `:latest` with moving target; no SBOM; ignoring base image updates.

### 7.9 Practical 9 — Multi-cluster monitoring (concept drill)

- **Proves:** You understand **federation vs central TSDB** (Thanos, Mimir, Cortex).
- **Steps:** Read architecture; explain how sidecar + object storage enables **global query**; know trade-offs (cost, cardinality, egress).
- **Success criteria:** Whiteboard HA Prometheus + long-term storage + global query path.

**Deeper:** [remaining_modules_summary.md](remaining_modules_summary.md), [MONITORING_SCANNING_CLOUD_TOOLS.md](MONITORING_SCANNING_CLOUD_TOOLS.md).

---

## Part 8 — Mock interviews

Mock interviews test **communication under ambiguity**, not only factual recall. Treat them like rehearsals with a **clear storyline**.

### 8.1 How to use mocks

1. **Pick one scenario** (design, debug, or trade-off) — don’t mix all three in one session.
2. **Timebox** 25–35 minutes; state **assumptions** early (scale, compliance, cloud).
3. **Draw** one diagram: users → ingress → services → data stores; or **failure** path for debug scenarios.
4. **Close with risks:** what you’d monitor, how you’d roll back, what you’d validate in staging.
5. **Review** against a simple rubric: **clarity**, **safety** (RBAC, secrets, blast radius), **operability** (observability, upgrades, runbooks).

### 8.2 Answer structure that works (STAR for behavioral; layered for system design)

- **Situation / Task:** Restate the problem; ask one or two clarifying questions if needed (SLO? multi-tenant? regulated data?).
- **Approach:** Start high-level, then drill down one layer at a time (network → platform → app → data).
- **Result:** How you’d verify success; what metrics prove it; what could go wrong.

For **system design**, a reliable skeleton is: **requirements → API/ingress → services → data → security → observability → rollout/rollback → cost.**

### 8.3 Scenario types (what to emphasize)

| Scenario | What to demonstrate |
|----------|---------------------|
| **Greenfield EKS platform** | Multi-AZ, node groups vs Fargate trade-offs, IRSA, cluster add-ons, GitOps (Flux/Argo), guardrails (OPA/Gatekeeper), cluster upgrades |
| **Production incident** | Triage order: **user impact → recent changes → metrics → traces → logs**; graceful rollback; postmortem habits |
| **Bare metal / self-managed** | etcd quorum, control plane HA, upgrades, backups, MetalLB/load balancing, storage classes |
| **Multi-cloud / hybrid** | Avoid lowest-common-denominator traps; **identity** (OIDC/workload identity), **secrets** (Vault vs cloud), **ingress** standardization |
| **Security hardening** | Pod Security Standards, NetworkPolicy, image provenance (signing), supply chain in CI, least-privilege RBAC |

### 8.4 Mini example — “Latency spike after deploy”

1. **Confirm symptom:** which SLO (p99 latency? error rate?) and scope (one service vs cluster-wide).
2. **Check change window:** deployment, ConfigMap, feature flag, canary vs full rollout.
3. **Narrow layer:** ingress/mesh timeouts, DB connection pool exhaustion, GC pause, downstream dependency.
4. **Decide:** rollback vs hotfix based on blast radius and data migration risk.
5. **Follow-up:** add **canary + automatic rollback**, better **SLI dashboards**, **load test** in staging.

**Deeper:** [Phase5-Interview-Practice.md](../Phase5-Interview-Practice.md), [INTERVIEW_PREP_COMPLETE_FINAL.md](INTERVIEW_PREP_COMPLETE_FINAL.md).

---

## Part 9 — Interview question bank (with answers)

Short answers you can expand in conversation. **More Q&A** exists in [core_resources_interview_qa.md](core_resources_interview_qa.md) and [networking_deep_dive.md](networking_deep_dive.md).

### Core resources

**Q1. Pod vs Deployment?**  
Pod is one-off or lowest level; Deployment manages replicas and rolling updates of identical Pods.

**Q2. When StatefulSet?**  
Stable identity, ordered rollout, per-replica storage — clustered stateful systems.

**Q3. What do init containers do?**  
Run to completion before app containers — bootstrap, migrations, wait-for dependencies.

### Networking

**Q4. ClusterIP vs NodePort vs LoadBalancer?**  
Internal only vs host port on all nodes vs cloud load balancer with external IP/DNS.

**Q5. What is a CNI?**  
Plugin that assigns Pod IPs and connects Pods across nodes; implements NetworkPolicy if supported.

### Advanced

**Q6. How does HPA differ from Cluster Autoscaler?**  
HPA scales **Pod replicas**; Cluster Autoscaler scales **nodes** when Pods cannot schedule.

**Q7. What is IRSA?**  
EKS OIDC trust maps Kubernetes ServiceAccount to IAM role so Pods get temporary AWS credentials without static keys.

### Prometheus / Grafana

**Q8. What is a time series?**  
Metric name + labels + timestamped values; Prometheus stores them for PromQL.

**Q9. What is recording rule?**  
Precomputed expensive queries, stored as new metrics for faster dashboards/alerts.

**Q10. Grafana data source types?**  
Prometheus, Loki, Tempo, CloudWatch, etc. — Grafana only visualizes; source holds data.

### Logging

**Q11. Why structured logs?**  
Parse fields (level, trace_id) for queries and correlation with metrics/traces.

**Q12. Loki vs Elasticsearch?**  
Loki indexes labels not full text (cheaper); ES full-text search (heavier cost).

### Tracing

**Q13. What is a span?**  
One unit of work in a trace; spans link via trace ID.

**Q14. W3C Trace Context?**  
Standard `traceparent` header so services propagate correlation IDs across mesh and HTTP.

### Service mesh

**Q15. Sidecar vs ambient mesh?**  
Sidecar: proxy per Pod; ambient: layer-4/7 path without always injecting per workload (product-specific).

**Q16. mTLS in mesh?**  
Encrypts service-to-service; identity from SPIFFE/SPIRE or mesh CA.

### Security

**Q17. Image pull secrets vs IRSA?**  
Image pull: registry auth; IRSA: AWS API auth from Pod.

**Q18. Pod Security Standards?**  
Built-in profiles (privileged, baseline, restricted) replacing deprecated PSP.

**Q19. OPA Gatekeeper vs admission webhook?**  
Gatekeeper packages OPA as K8s policies with CRDs and audit.

**Q20. NetworkPolicy default?**  
If no policies: usually open; with policies: depends on CNI — often need explicit DNS allow.

### Cloud-native

**Q21. Secrets Manager vs SSM Parameter Store?**  
Secrets rotation + secret-centric vs broader config parameters; different cost/APIs.

**Q22. KMS envelope encryption?**  
Data key encrypted by KMS master key; limits KMS calls.

**Q23. External Secrets Operator benefit?**  
GitOps-friendly: source of truth in Vault/cloud; sync into K8s Secret objects.

**Q24. Why avoid plaintext secrets in Git?**  
History is forever; use Sealed Secrets, SOPS, or ESO + external store.

**Q25. Multi-account AWS pattern?**  
Org units + SCPs guardrails; workload accounts assume roles into shared services.

**Q26. Vault vs cloud secrets?**  
Vault: multi-cloud, dynamic secrets, PKI; cloud native: simpler if all-in on one cloud.

### Extra rapid-fire (scheduling & operations)

**Q27. Requests vs limits?**  
Requests drive scheduling (kube-scheduler “fits” on node); limits cap max usage; exceeding CPU may throttle; exceeding memory can OOMKill.

**Q28. What is a taint / toleration?**  
Taint repels Pods unless they tolerate it; used for dedicated node pools (GPU, system nodes) or cordoning bad nodes.

**Q29. PodDisruptionBudget?**  
Limits **voluntary** disruptions (drain, rollout) so a minimum number of Pods stay available—pairs with HA expectations.

**Q30. ResourceQuota / LimitRange?**  
Quota caps aggregate use per namespace; LimitRange sets defaults/min/max for individual objects—prevents one team starving others.

**Q31. What does Cluster Autoscaler do?**  
Adds/removes **nodes** when Pods are unschedulable or nodes underutilized; different from HPA which scales **Pods**.

**Q32. Admission controllers?**  
Mutating/validating webhooks (OPA Gatekeeper, cert-manager, Istio) intercept API requests before objects persist.

---

## After this document

- **Drill:** Re-read Parts 1–3 until you can whiteboard without notes.
- **Platform:** Part 4 + `eks_architecture_guide.md` for your target cloud.
- **Ops:** Parts 5–6 + monitoring file for on-call style questions.
- **Practice:** Part 7 items on a real cluster; Part 8–9 with a peer.

---

*Aligned with [k8s_study_guide.md](k8s_study_guide.md) Parts 1–9. For encyclopedic depth per topic, follow the “Deeper” links in each part.*
