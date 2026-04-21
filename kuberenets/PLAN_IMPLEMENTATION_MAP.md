# Plan ↔ study materials mapping

This file ties **todo IDs from the Kubernetes Interview Preparation Plan** (frontmatter in the plan document) to **concrete files** under:

`/home/frontier/devops-interview-prep/aws-devops/kuberenets`

Do **not** edit the plan file itself; use this map for navigation.

| Todo ID | Plan topic | Primary material |
|---------|------------|----------------|
| `core_resources` | Pod, Deployment, StatefulSet, DaemonSet, Job, Service, Ingress | `core_resources_guide.md`, `core_resources_guide.yaml`, `core_resources_interview_qa.md` |
| `storage_volumes` | PV, PVC, StorageClass, EBS/EFS/Azure | `storage_concepts_guide.md` |
| `networking_deep_dive` | CNI, CoreDNS, kube-proxy | `networking_deep_dive.md` |
| `network_policies` | NetworkPolicy scenarios | `networking_deep_dive.md` |
| `crds_operators` | CRDs, operators | `crds_operators_guide.md` |
| `eks_architecture` | EKS | `eks_architecture_guide.md` |
| `aks_architecture` | AKS | `aks_architecture_guide.md` |
| `bare_metal` | Bare metal / kubeadm | `bare_metal_kubernetes_guide.md` |
| `multi_arch` | x64, ARM64, manifests | `remaining_modules_summary.md` |
| `scaling_ha` | HPA, CA, Karpenter, PDB | `remaining_modules_summary.md` |
| `security_rbac` | RBAC, PSS, secrets | `remaining_modules_summary.md` |
| `observability` | Logs, metrics, traces (overview) | `remaining_modules_summary.md`, `MONITORING_SCANNING_CLOUD_TOOLS.md` |
| `monitoring_tools` | Prometheus, Grafana, **Alertmanager**, Loki, ELK/Kibana | `MONITORING_SCANNING_CLOUD_TOOLS.md` (Parts 1–2, **Appendix** Alertmanager) |
| `tracing_stack` | Jaeger, **OpenTelemetry**, correlation | `MONITORING_SCANNING_CLOUD_TOOLS.md` (Jaeger section + **Appendix** OTel) |
| `service_mesh` | Istio, Cilium, Hubble | `MONITORING_SCANNING_CLOUD_TOOLS.md` Part 2 |
| `security_scanning` | Trivy, Snyk, Falco, **OPA/Gatekeeper**, **kube-bench** | `MONITORING_SCANNING_CLOUD_TOOLS.md` Part 3 + **Appendix** OPA/kube-bench |
| `aws_tools` | Secrets Manager, Parameter Store, CloudWatch, X-Ray, ECR, GuardDuty, Security Hub | `MONITORING_SCANNING_CLOUD_TOOLS.md` Part 4 |
| `azure_tools` | Key Vault, App Insights, Log Analytics, ACR, Defender (replaces legacy “Security Center” naming) | `MONITORING_SCANNING_CLOUD_TOOLS.md` Part 5 |
| `vault_secrets` | HashiCorp Vault | `MONITORING_SCANNING_CLOUD_TOOLS.md` Part 6 |
| `helm_gitops` | Helm, ArgoCD | `remaining_modules_summary.md`, `CICD_DEPLOYMENT_TOOLS.md` |
| `hands_on_labs` | Practicals checklist | `remaining_modules_summary.md` |
| `workspace_study` | Workspace pointers | `INTERVIEW_PREP_COMPLETE_FINAL.md` (workspace refs), plan doc (repo paths) |
| `mock_interview` | Mock scenarios | `INTERVIEW_PREP_COMPLETE_FINAL.md`, `INDEX_COMPLETE_STUDY_PACKAGE.md` |

**Single-file deep reference for tools:** `MONITORING_SCANNING_CLOUD_TOOLS.md`  
**CI/CD (ArgoCD, Jenkins, GitHub Actions, Azure Pipelines):** `CICD_DEPLOYMENT_TOOLS.md`

---

*Aligned with plan implementation: 2026-04-21*
