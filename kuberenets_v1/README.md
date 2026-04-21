# Kubernetes interview preparation — `kuberenets_v1`

Modular, **CKA/CKAD-style** depth: architecture, core APIs, networking, storage, security, operations, themed Q&A, and hands-on labs.  
**Prerequisite:** working `kubectl` and a cluster (kind, minikube, k3d, or cloud-managed). Container basics live in the parent repo’s Phase 2 doc (linked below).

---

## Prerequisites

| Need | Notes |
|------|--------|
| `kubectl` | v1.28+ recommended; match minor version to cluster when possible |
| Cluster | [kind](https://kind.sigs.k8s.io/), [minikube](https://minikube.sigs.k8s.io/), [k3d](https://k3d.io/), or EKS/GKE/AKS |
| Shell | bash/zsh; optional: `helm`, `k9s` |

**Related repo material:** [Phase2-Containerization-Orchestration.md](../Phase2-Containerization-Orchestration.md) (Docker, ECS, EKS intro).

---

## Study map (suggested order)

| Day | Read | Practice |
|-----|------|----------|
| **1** | [01-architecture-and-components.md](01-architecture-and-components.md) | Skim cluster with `kubectl get --raw /healthz`, `kubectl get nodes -o wide` |
| **2** | [02-workloads-and-scheduling.md](02-workloads-and-scheduling.md) | Labs 1–3, 13 (affinity/taints) in [08-hands-on-labs.md](08-hands-on-labs.md) |
| **3** | [03-networking-and-services.md](03-networking-and-services.md) | Labs 4–6 |
| **4** | [04-storage-and-config.md](04-storage-and-config.md) | Labs 7–8 |
| **5** | [05-security-rbac-and-policy.md](05-security-rbac-and-policy.md) | Lab 9 |
| **6** | [06-operations-troubleshooting-gitops.md](06-operations-troubleshooting-gitops.md) | Labs 10–12 |
| **7** | [09-crds-and-operators.md](09-crds-and-operators.md) | Lab 14 in [08-hands-on-labs.md](08-hands-on-labs.md) (optional) |
| **8** | [07-interview-questions.md](07-interview-questions.md) | Timed verbal drill + repeat weak labs |

**Fast path (3 days):** 01 → 02 → 03, then 07 + Labs 1–6, 9–10. Add **09** when interviewing for platform / operator work.

---

## File index

| File | Topics |
|------|--------|
| [01-architecture-and-components.md](01-architecture-and-components.md) | Control plane, nodes, etcd, request flow, HA |
| [02-workloads-and-scheduling.md](02-workloads-and-scheduling.md) | Pods, Deployments, StatefulSets, Jobs, probes, affinity, PDB, HPA |
| [03-networking-and-services.md](03-networking-and-services.md) | Service types, Endpoints, DNS, Ingress, Gateway API, NetworkPolicy |
| [04-storage-and-config.md](04-storage-and-config.md) | Volumes, PV/PVC, StorageClass, CSI, ConfigMap, Secret |
| [05-security-rbac-and-policy.md](05-security-rbac-and-policy.md) | AuthN/Z, RBAC, SA, PSS, admission, imagePullSecrets |
| [06-operations-troubleshooting-gitops.md](06-operations-troubleshooting-gitops.md) | kubectl, debug, quotas, Helm, GitOps, upgrades |
| [09-crds-and-operators.md](09-crds-and-operators.md) | CRDs, custom resources, validation, operators, RBAC |
| [07-interview-questions.md](07-interview-questions.md) | Themed Q&A with sample answers |
| [08-hands-on-labs.md](08-hands-on-labs.md) | CKA-style tasks + solution outlines |

---

## How to use this pack

1. **Theory:** Read each numbered chapter; complete the **Quick recap** and try the **Interview prompts** aloud.
2. **Drill:** After chapters 02–06 (and **09** if relevant), do the matching labs in `08-hands-on-labs.md` without peeking at solutions first.
3. **Interview:** Use `07-interview-questions.md` for mock sessions; explain diagrams from `01` on a whiteboard.

---

## Scope and “everything Kubernetes”

This pack is **interview-complete** for core Kubernetes plus an **CRD/operator** chapter (**09**). For longer EKS/service-mesh/multi-cluster notes, see `../kuberenets/` and [MASTER_COVERAGE.md](../MASTER_COVERAGE.md).

---

## Parent navigation

- [MASTER_COVERAGE.md](../MASTER_COVERAGE.md) — full repo map  
- [README-INTERVIEW-PREP.md](../README-INTERVIEW-PREP.md) — 5-phase program  
