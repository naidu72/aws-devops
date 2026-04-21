# Kubernetes interview prep — progress summary

**Location:** `/home/frontier/devops-interview-prep/aws-devops/kuberenets`  
**Navigation:** See [README.md](README.md) and [INDEX_COMPLETE_STUDY_PACKAGE.md](INDEX_COMPLETE_STUDY_PACKAGE.md).

---

## Status: all planned modules covered

This folder supersedes older drafts that referenced `/tmp/...`. Every guide listed below exists **in this directory**.

| Module | Topics | Primary files |
|--------|--------|-----------------|
| Core resources | Pod, Deployment, StatefulSet, DaemonSet, Job, Service, Ingress | [core_resources_interview_qa.md](core_resources_interview_qa.md), [core_resources_guide.yaml](core_resources_guide.yaml) |
| Storage | PV, PVC, StorageClass, cloud volumes | [storage_concepts_guide.md](storage_concepts_guide.md) |
| Networking | CNI, DNS, kube-proxy, NetworkPolicy | [networking_deep_dive.md](networking_deep_dive.md) |
| CRDs & operators | CRD, webhooks, operators | [crds_operators_guide.md](crds_operators_guide.md) |
| EKS | Control plane, node groups, IRSA, add-ons | [eks_architecture_guide.md](eks_architecture_guide.md) |
| AKS | Node pools, identity, networking | [aks_architecture_guide.md](aks_architecture_guide.md) |
| Bare metal | kubeadm, HA, storage, load balancing | [bare_metal_kubernetes_guide.md](bare_metal_kubernetes_guide.md) |
| Advanced summary | Multi-arch, scaling, HA, security, Helm, GitOps | [remaining_modules_summary.md](remaining_modules_summary.md) |
| Observability & cloud tools | Prometheus, Loki, Istio, scanning, AWS/Azure/Vault | [MONITORING_SCANNING_CLOUD_TOOLS.md](MONITORING_SCANNING_CLOUD_TOOLS.md) |
| CI/CD | ArgoCD, Jenkins, GitHub Actions, Azure DevOps | [CICD_DEPLOYMENT_TOOLS.md](CICD_DEPLOYMENT_TOOLS.md) |
| Meta | Study plan, checklists, completion summary | [k8s_study_guide.md](k8s_study_guide.md), [INTERVIEW_PREP_COMPLETE_FINAL.md](INTERVIEW_PREP_COMPLETE_FINAL.md), [INDEX_COMPLETE_STUDY_PACKAGE.md](INDEX_COMPLETE_STUDY_PACKAGE.md) |

---

## File inventory (paths relative to this folder)

| # | File | Notes |
|---|------|--------|
| 1 | `README.md` | Master navigation |
| 2 | `INDEX_COMPLETE_STUDY_PACKAGE.md` | Index and study roadmap |
| 3 | `k8s_study_guide.md` | Timeline and structure |
| 4 | `core_resources_guide.yaml` | Compact YAML reference (was missing; now added) |
| 5 | `core_resources_interview_qa.md` | Core resource Q&A |
| 6 | `storage_concepts_guide.md` | Storage deep dive |
| 7 | `networking_deep_dive.md` | Networking (replaces old name `networking_guide.md`) |
| 8 | `crds_operators_guide.md` | CRDs and operators |
| 9 | `eks_architecture_guide.md` | EKS |
| 10 | `aks_architecture_guide.md` | AKS |
| 11 | `bare_metal_kubernetes_guide.md` | On-prem |
| 12 | `remaining_modules_summary.md` | Multi-arch, scaling, security, observability summary |
| 13 | `MONITORING_SCANNING_CLOUD_TOOLS.md` | Monitoring, scanning, cloud tools |
| 14 | `CICD_DEPLOYMENT_TOOLS.md` | CI/CD tools |
| 15 | `INTERVIEW_PREP_COMPLETE_FINAL.md` | Final checklist |
| 16 | `INTERVIEW_PREP_COMPLETE.md` | Older snapshot (paths updated) |
| 17 | `study_summary.md` | This file |

---

## What was wrong before (fixed)

- **Stale `/tmp/` paths:** Materials were generated under `/tmp` and copied here; links still pointed at `/tmp`, so files looked “missing.” All indexes now use **this folder**.
- **Missing `core_resources_guide.yaml`:** Referenced in older summaries but not shipped; added a **compact** multi-resource reference (expand with `core_resources_interview_qa.md` for depth).
- **Wrong networking filename:** Summaries mentioned `networking_guide.md`; the actual file is **`networking_deep_dive.md`**.

---

## Suggested next actions (self-study)

1. Open [README.md](README.md) and pick one track (platform vs networking vs observability).
2. Cross-check interview readiness with [INTERVIEW_PREP_COMPLETE_FINAL.md](INTERVIEW_PREP_COMPLETE_FINAL.md).
3. For hands-on, use manifests from [core_resources_guide.yaml](core_resources_guide.yaml) in a test namespace.

---

*Last aligned with folder contents: 2026-04-21*
