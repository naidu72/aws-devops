# 07 — Interview questions (themed)

Practice **out loud** in 2–3 minutes per answer unless noted. Sample bullets are hints—not scripts to memorize.

---

## Architecture and control plane

**Q1. Explain Kubernetes architecture at a high level.**  
**A:** Client (kubectl/controller) talks to **kube-apiserver**. State is stored in **etcd**. **Scheduler** assigns Pods to nodes. **Controller manager** reconciles Deployments, ReplicaSets, etc. On each node, **kubelet** runs containers via the **container runtime**; **kube-proxy** (or CNI) implements Services. **Cloud controller manager** integrates with cloud load balancers and routes when applicable.

**Q2. What is etcd and why does quorum matter?**  
**A:** etcd is the consistent, distributed store for Kubernetes objects. Writes need a **majority** of members (quorum). Losing quorum **blocks writes** (cluster may become read-only or unhealthy), risking control plane outage. Backups and odd-member clusters (3, 5) are standard.

**Q3. What happens when the API server is unavailable?**  
**A:** No cluster changes (apply, scale, delete). Existing workloads **keep running** on nodes; kubelet may cache specs but cannot report status indefinitely; operators lose visibility and control until API returns.

**Q4. How does version skew affect upgrades?**  
**A:** API server should be **newest** minor; kubelet can be **up to two minors older** (check current policy); kubectl can be one minor newer or older. Skew prevents incompatible API/feature assumptions.

---

## Workloads and scheduling

**Q5. Deployment vs StatefulSet vs DaemonSet?**  
**A:** **Deployment** — stateless, rolling updates, arbitrary Pod names. **StatefulSet** — stable identity, ordered rollout, often PVC per replica. **DaemonSet** — one Pod per eligible node (agents).

**Q6. Liveness vs readiness probes?**  
**A:** **Liveness** — kubelet **restarts** failing container. **Readiness** — Pod removed from **Service endpoints** while failing. Wrong probe choice causes traffic to not-ready Pods or unnecessary restarts.

**Q7. How does the scheduler pick a node?**  
**A:** **Filtering** (resources, selectors, taints, topology) then **scoring** (spread, affinity, resource fit). First-class concepts: **requests/limits**, **taints/tolerations**, **node affinity**, **Pod topology spread**.

**Q8. What is a PodDisruptionBudget?**  
**A:** Limits **voluntary** evictions (drain, rolling update) so a minimum availability or maximum unavailable is respected; does **not** stop involuntary failures.

**Q8b. Node affinity vs `nodeSelector`?**  
**A:** `nodeSelector` is a simple **AND** of required labels. **Node affinity** supports **hard vs soft** rules, richer **operators** (`NotIn`, `Exists`, …), and **OR** across `nodeSelectorTerms` (any term can match).

**Q8c. What is pod anti-affinity and a typical `topologyKey`?**  
**A:** Rules that **avoid** placing Pods near other Pods matching a `labelSelector`. `topologyKey: kubernetes.io/hostname` spreads across **nodes**; `topology.kubernetes.io/zone` spreads across **AZs**. Hard (`required`) rules can block scheduling if capacity/topology is insufficient.

**Q8d. Topology spread constraints vs pod anti-affinity?**  
**A:** **Spread** enforces **skew** (difference in count) between topology domains—good for **even** replica distribution. **Anti-affinity** is **pairwise** “don’t co-locate with same label.” Both can be combined; spread is often easier for zone balancing.

**Q8e. Taint effects `NoSchedule` vs `NoExecute`? When use `tolerationSeconds`?**  
**A:** **NoSchedule** blocks new Pods without toleration. **NoExecute** also **evicts** running Pods without toleration. **tolerationSeconds** (on a **NoExecute** toleration) limits how long a Pod may stay bound after a taint appears—common for **node-not-ready** style tolerations.

**Q8f. Why do DaemonSet Pods appear on tainted nodes when Deployments do not?**  
**A:** The DaemonSet controller adds **default tolerations** so node-level agents (logging, monitoring, CNI) still run on tainted/NotReady nodes per policy.

---

## Networking and Services

**Q9. How does a ClusterIP Service work?**  
**A:** A virtual IP targets **EndpointSlices** of healthy Pod IPs matching selectors. **kube-proxy** or CNI programs rules to load-balance traffic. **CoreDNS** resolves `svc.namespace.svc.cluster.local`.

**Q10. Headless Service — when and why?**  
**A:** `clusterIP: None`. DNS returns **Pod records** directly. Used for StatefulSets, peer discovery, or when you want client-side load balancing.

**Q11. Ingress vs LoadBalancer Service?**  
**A:** **LoadBalancer** is typically L4 cloud LB to one Service. **Ingress** is L7 HTTP routing to multiple Services; requires an **Ingress controller**.

**Q12. NetworkPolicy default behavior?**  
**A:** With most CNIs, **no policies** means **allow all**. Once policies exist, behavior depends on CNI—many patterns use explicit **default deny** ingress/egress plus allow rules. Always name your CNI (Calico, Cilium, etc.).

---

## Storage and configuration

**Q13. PVC Pending — what do you check?**  
**A:** `describe pvc` — no matching PV, StorageClass provisioner failed, quota, **WaitForFirstConsumer** with no Pod scheduled, wrong access mode or size.

**Q14. ConfigMap vs Secret?**  
**A:** Both inject config; **Secret** is for sensitive data (still not encrypted in etcd without encryption-at-rest). Prefer mounting files over env for secrets; integrate external secret managers for rotation.

**Q15. Dynamic provisioning flow?**  
**A:** PVC references **StorageClass** → **provisioner** creates **PV** → binds PVC → kubelet attaches/mounts via CSI on scheduled node.

---

## Security and RBAC

**Q16. Role vs ClusterRole?**  
**A:** **Role** is namespace-scoped. **ClusterRole** is cluster-wide or used via **RoleBinding** to grant cluster permissions **into** one namespace (common pattern for aggregated permissions).

**Q17. How do applications authenticate to the API?**  
**A:** **ServiceAccount** + mounted **projected token** (short-lived) or bound token; RBAC restricts verbs/resources. For human users, OIDC/certs.

**Q18. What replaced PodSecurityPolicy?**  
**A:** **Pod Security Standards** enforced by namespace labels and/or admission (built-in admission in 1.23+, or policies via Kyverno/Gatekeeper). PSP removed in 1.25.

---

## Operations and troubleshooting

**Q19. Pod in CrashLoopBackOff — steps?**  
**A:** `describe` (events), `logs` and `logs --previous`, verify command, env, ConfigMap/Secret mounts, probes too aggressive, resource limits OOM.

**Q20. How do you safely maintain a node?**  
**A:** `cordon` (no new Pods), `drain` with PDB respected, patch/replace node, `uncordon`. DaemonSets need `--ignore-daemonsets`.

**Q21. How would you find deprecated APIs before upgrade?**  
**A:** Audit server logs, **pluto/kubent**, test in lower env, review release notes; migrate to `v1` / stable APIs.

**Q22. Helm benefit over raw YAML?**  
**A:** Templating, chart versioning, **release history** and rollback, dependency packaging—trade-off: complexity and state in cluster (release secrets/configmaps).

---

## Scenario-style (senior)

**Q23. “Traffic is flaky only during deploys.”**  
**A:** Check **readiness** probes, **maxUnavailable** / **maxSurge**, **PDB**, **preStop** hooks / **terminationGracePeriodSeconds**, client retry behavior, **Ingress** health checks.

**Q24. “One namespace cannot schedule Pods.”**  
**A:** **ResourceQuota** full, **LimitRange** violations, **Priority** preemption, **taints** with no toleration, **PVC** not bound for StatefulSet.

**Q25. “We need multi-tenant isolation.”**  
**A:** Namespaces + **RBAC** + **NetworkPolicy** + **ResourceQuota** + **PSS**; stronger isolation may need separate clusters or hypervisor-level boundaries; mention **audit** and **admission webhooks**.

---

## CRDs and operators

**Q26. What is a CRD and what does it not do by itself?**  
**A:** A **CustomResourceDefinition** registers a new API **Kind** with the API server (schema, scope, versions). It does **not** reconcile cluster state—unless a **controller/operator** watches those objects and creates Pods, Services, etc.

**Q27. Why use `subresources.status` on a CRD?**  
**A:** Separates **`spec`** (user intent) from **`status`** (observed state). Controllers patch `/status`; RBAC can allow `update/status` without granting full `update` on the object—cleaner and safer.

**Q28. How do you discover installed custom APIs?**  
**A:** `kubectl api-resources`, `kubectl get crd`, `kubectl explain <kind>.<group>`.

**Q29. What is an operator vs a plain controller?**  
**A:** **Controller** is the reconcile loop pattern. **Operator** usually means domain-specific automation packaged for lifecycle (install, upgrade, backup) around one or more CRDs—often Helm/OCI bundles or OLM.

**Q30. What blocks deletion of a custom resource?**  
**A:** **Finalizers** on `metadata.finalizers`—controller must remove them after cleanup; also normal protections (foreground deletion, dependents).

---

## Quick drill checklist

- Draw architecture from memory (01).
- Explain a rolling update + rollback (02).
- Whiteboard: node affinity vs pod anti-affinity vs topology spread; name all three **taint effects** (02).
- Explain CRD → CR instance → controller → child objects (09).
- Trace a DNS query to a Service and to a Pod IP (03).
- Recite RBAC objects and one binding example (05).
