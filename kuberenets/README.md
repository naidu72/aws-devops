# Kubernetes interview prep — file index

> **Entire repo (phases + this folder + lab notes):** [`MASTER_COVERAGE.md`](../MASTER_COVERAGE.md)

**Folder:** `/home/frontier/devops-interview-prep/aws-devops/kuberenets`

All markdown paths below are **relative to this directory** (same folder as this `README.md`).

## Start here

| File | Purpose |
|------|---------|
| [k8s_study_guide_PARTS_1-9_DETAILED.md](k8s_study_guide_PARTS_1-9_DETAILED.md) | **Teaching narrative for Parts 1–9** (same outline as [k8s_study_guide.md](k8s_study_guide.md)) |
| [PLAN_IMPLEMENTATION_MAP.md](PLAN_IMPLEMENTATION_MAP.md) | **Maps plan todo IDs → files/sections** (use with the official plan doc) |
| [INDEX_COMPLETE_STUDY_PACKAGE.md](INDEX_COMPLETE_STUDY_PACKAGE.md) | Master index, topic checklist, study plan |
| [study_summary.md](study_summary.md) | Module status and file map (kept in sync with this README) |
| [INTERVIEW_PREP_COMPLETE_FINAL.md](INTERVIEW_PREP_COMPLETE_FINAL.md) | Final summary and interview checklists |

## Core Kubernetes

| File | Purpose |
|------|---------|
| [k8s_study_guide.md](k8s_study_guide.md) | Structured study plan |
| [core_resources_interview_qa.md](core_resources_interview_qa.md) | Interview Q&A (core resources) |
| [core_resources_guide.yaml](core_resources_guide.yaml) | Multi-doc YAML to **`kubectl apply`** (Namespace, Pod, Deployment, Service, Ingress, StatefulSet, DaemonSet, Job, CronJob) |
| [core_resources_guide.md](core_resources_guide.md) | Same manifests explained in Markdown (copy-paste friendly) |
| [storage_concepts_guide.md](storage_concepts_guide.md) | PV, PVC, StorageClass, volume types |
| [networking_deep_dive.md](networking_deep_dive.md) | CNI, DNS, kube-proxy, NetworkPolicy |

## Platforms

| File | Purpose |
|------|---------|
| [eks_architecture_guide.md](eks_architecture_guide.md) | AWS EKS, IRSA, add-ons |
| [aks_architecture_guide.md](aks_architecture_guide.md) | Azure AKS, identity, networking |
| [bare_metal_kubernetes_guide.md](bare_metal_kubernetes_guide.md) | kubeadm, HA, MetalLB, on-prem |

## Advanced topics

| File | Purpose |
|------|---------|
| [crds_operators_guide.md](crds_operators_guide.md) | CRDs, operators, webhooks |
| [remaining_modules_summary.md](remaining_modules_summary.md) | Multi-arch, scaling, security, Helm, GitOps (summary) |

## Observability, security, and cloud tools

| File | Purpose |
|------|---------|
| [MONITORING_SCANNING_CLOUD_TOOLS.md](MONITORING_SCANNING_CLOUD_TOOLS.md) | Prometheus, Grafana, Loki, Istio, Trivy, AWS/Azure tools, Vault |

## CI/CD

| File | Purpose |
|------|---------|
| [CICD_DEPLOYMENT_TOOLS.md](CICD_DEPLOYMENT_TOOLS.md) | ArgoCD, Jenkins, GitHub Actions, Azure Pipelines |

## Legacy / alternate summaries

| File | Purpose |
|------|---------|
| [INTERVIEW_PREP_COMPLETE.md](INTERVIEW_PREP_COMPLETE.md) | Older progress doc (paths updated to this folder) |

---

**Note:** Older docs used `/tmp/...` paths from the original generation environment. Everything now lives under this folder permanently.
