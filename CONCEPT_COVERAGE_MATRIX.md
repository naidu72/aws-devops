# Concept coverage matrix — what exists vs what is light / missing

**Repo:** `/home/frontier/devops-interview-prep/aws-devops`  
**Companion:** [MASTER_COVERAGE.md](MASTER_COVERAGE.md) (navigation + lab notes)

This file is the answer to: *“I read the summaries—are any important DevOps/Kubernetes concepts not actually covered?”*

**Legend**

| Symbol | Meaning |
|--------|---------|
| **Deep** | Dedicated section(s); interview-ready if you study the file |
| **Summary** | Covered in a summary module (often `remaining_modules_summary.md` or a Phase overview) — enough for awareness, not always hands-on |
| **Mention** | Named once or in a checklist; you must supplement elsewhere |
| **Gap** | Not meaningfully in this repo — use external resource (link below) |

---

## A. AWS platform (Phases + `Phase1`)

| Concept | Depth | Where in this repo |
|---------|--------|---------------------|
| IAM users/roles/policies, STS, cross-account | **Deep** | `Phase1-AWS-Fundamentals.md` §1 |
| EC2, ASG, ELB/ALB/NLB, EBS, IMDSv2 | **Deep** | `Phase1-AWS-Fundamentals.md` §1.2 |
| S3 (classes, encryption, replication, versioning) | **Deep** | `Phase1-AWS-Fundamentals.md` §1.3 |
| RDS, DynamoDB | **Deep** | `Phase1-AWS-Fundamentals.md` §1.4 |
| VPC, subnets, NAT, endpoints, SG vs NACL | **Deep** | `Phase1-AWS-Fundamentals.md` §1.5 |
| Route53, CloudFront | **Deep** | `Phase1-AWS-Fundamentals.md` §1.5 |
| Hands-on AWS CLI labs | **Deep** | `Phase1-Hands-On-Labs.md` |
| **AWS Organizations, SCPs, Control Tower, multi-account guardrails** | **Gap** | AWS whitepapers / AWS Organizations docs; add to Phase1 if you need interviews on landing zones |
| **SQS / SNS / EventBridge** (patterns, DLQ, fan-out) | **Mention** | IAM/resource examples in Phase1; **not** a dedicated messaging chapter |
| **API Gateway** (HTTP vs REST vs WebSocket) | **Mention** | Sparse vs Lambda in phases; **Gap** for deep REST API management |
| **MSK / Kafka on AWS** | **Gap** | Not a first-class topic here; see AWS MSK docs + `kuberenets/crds_operators_guide.md` (Kafka operator mention only) |

---

## B. Containers & Kubernetes core

| Concept | Depth | Where in this repo |
|---------|--------|---------------------|
| Docker images, layers, basic security | **Deep** | `Phase2-Containerization-Orchestration.md` |
| Pod, Deployment, Service, Ingress, SS, DS, Job | **Deep** | `kuberenets/core_resources_*.md` + YAML |
| Resource requests/limits, QoS (implicit in Q&A) | **Summary** | `core_resources_interview_qa.md`, `remaining_modules_summary.md` |
| RBAC | **Summary** | `remaining_modules_summary.md` (security), Phase4 |
| **Pod Security Admission / PSS** | **Summary** | `remaining_modules_summary.md` |
| Storage (PV/PVC/SC) | **Deep** | `kuberenets/storage_concepts_guide.md` |
| Networking: CNI compare, NetworkPolicy, kube-proxy, EndpointSlices | **Deep** | `kuberenets/networking_deep_dive.md` |
| **Kubernetes Gateway API** (Gateway/HTTPRoute vs Ingress) | **Gap** | [gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io/) — increasingly common in interviews |
| **Windows nodes / mixed OS** | **Gap** | AKS/EKS docs for Windows node pools |

---

## C. Cloud Kubernetes (managed)

| Concept | Depth | Where in this repo |
|---------|--------|---------------------|
| EKS: control plane, node groups, IRSA, add-ons, VPC CNI notes | **Deep** | `kuberenets/eks_architecture_guide.md` |
| AKS: identity, node pools, networking | **Deep** | `kuberenets/aks_architecture_guide.md` |
| Bare metal / kubeadm / MetalLB | **Deep** | `kuberenets/bare_metal_kubernetes_guide.md` |
| **GKE** (Autopilot, workload identity) | **Gap** | Google GKE docs if multi-cloud interview |

---

## D. GitOps, CI/CD, IaC

| Concept | Depth | Where in this repo |
|---------|--------|---------------------|
| Pipeline concepts, blue/green, canary, rolling | **Deep** | `Phase3-CICD-Automation.md` |
| Argo CD, Jenkins, GitHub Actions, Azure DevOps | **Deep** | `kuberenets/CICD_DEPLOYMENT_TOOLS.md` |
| Terraform / CloudFormation (as automation) | **Summary** | `Phase3-CICD-Automation.md`, `Phase4-Advanced-Topics.md` touches |
| **Terragrunt** (DRY multi-env) | **Gap** | HashiCorp Terragrunt docs; optional sibling `terraform-iac/` outside this repo |
| **Argo Rollouts** (canary analysis) | **Mention** | Istio canary in phases; **Gap** for Rollouts CRD specifics |

---

## E. Observability & reliability

| Concept | Depth | Where in this repo |
|---------|--------|---------------------|
| Prometheus, Grafana, PromQL snippets | **Deep** | `kuberenets/MONITORING_SCANNING_CLOUD_TOOLS.md` Part 1 |
| Loki, ELK positioning | **Deep** | Same |
| Jaeger tracing | **Deep** | Same Part 1 / 2 |
| **OpenTelemetry** (collector, OTLP, vs Jaeger) | **Deep** | `MONITORING_SCANNING_CLOUD_TOOLS.md` **Appendix** (~line 1807+) |
| CloudWatch, alarms, SNS hook example | **Summary** | `MONITORING_SCANNING_CLOUD_TOOLS.md` Part 4, Phase4 |
| **SLO / SLI / error budgets** (formal) | **Summary** | `Phase4-Advanced-Topics.md` observability; refine with Google SRE book if asked deeply |
| **Real User Monitoring (RUM)** | **Gap** | Vendor docs (Datadog RUM, CloudWatch RUM) |
| **AWS Distro for OpenTelemetry (ADOT)** | **Mention** | OTel appendix; **Gap** for AWS-managed collector deep dive |

---

## F. Security & compliance

| Concept | Depth | Where in this repo |
|---------|--------|---------------------|
| Image scan: Trivy, Snyk | **Deep** | `MONITORING_SCANNING_CLOUD_TOOLS.md` Part 3 |
| Runtime: Falco | **Deep** | Same |
| Policy: **OPA Gatekeeper** | **Deep** | Same appendix |
| **kube-bench** (CIS) | **Deep** | Same appendix |
| Secrets: AWS Secrets Manager, SSM, KMS examples | **Deep** | `MONITORING_SCANNING_CLOUD_TOOLS.md` Part 4 |
| **External Secrets Operator** | **Mention** | Checklist in `INTERVIEW_PREP_COMPLETE_FINAL.md`; **Gap** for install YAML walkthrough |
| **Kyverno** (policy alternative) | **Gap** | [kyverno.io](https://kyverno.io/) — compare to Gatekeeper in one page of notes |
| **Cosign / Sigstore / SBOM** (supply chain) | **Gap** | Trivy scans images; **not** full sigstore/cosign signing flow |
| **SPIFFE / SPIRE** | **Gap** | CNCF docs if zero-trust identity interview |

---

## G. Service mesh & traffic

| Concept | Depth | Where in this repo |
|---------|--------|---------------------|
| Istio basics, traffic, mTLS mention | **Deep** | `MONITORING_SCANNING_CLOUD_TOOLS.md` §2.1 Istio |
| Cilium / Hubble | **Deep** | `MONITORING_SCANNING_CLOUD_TOOLS.md` §2.2, `networking_deep_dive.md` |
| **Linkerd** (lighter mesh) | **Gap** | [linkerd.io](https://linkerd.io/) comparison to Istio |
| **Ingress vs mesh Gateway troubleshooting** | **Deep** | [MASTER_COVERAGE.md](MASTER_COVERAGE.md) §5 (404, IST0145, VS order) — *lab-hardened* |

---

## H. Data, messaging, persistence on K8s

| Concept | Depth | Where in this repo |
|---------|--------|---------------------|
| StatefulSet + storage patterns | **Deep** | `storage_concepts_guide.md`, `core_resources_*` |
| Operators, Kafka/Rabbit **mentioned** as use cases | **Mention** | `crds_operators_guide.md` |
| **Running Kafka / MSK / eventing architecture** | **Gap** | Confluent/AWS MSK guides |
| **Postgres/MySQL on K8s** (operators) | **Summary** | StatefulSet + operator pattern; **Gap** for Crunchy/CloudNative-PG specifics |

---

## I. Backup, DR, upgrades

| Concept | Depth | Where in this repo |
|---------|--------|---------------------|
| DR, backups (conceptual) | **Summary** | `Phase4-Advanced-Topics.md`, `remaining_modules_summary.md` |
| **etcd backup / restore** (self-managed) | **Summary** | `bare_metal_kubernetes_guide.md`, `remaining_modules_summary.md` checklist |
| **Velero** (cluster backup) | **Mention** | Checklist level; **Gap** for install/restore lab |
| **EKS control plane / addon upgrades** | **Summary** | `eks_architecture_guide.md` |

---

## J. “Hidden” gems already in repo (easy to miss)

| Topic | Why it matters | File |
|--------|----------------|------|
| OpenTelemetry appendix | Often asked vs “Jaeger only” | `MONITORING_SCANNING_CLOUD_TOOLS.md` (end) |
| OPA Gatekeeper YAML | Policy-as-code interviews | Same |
| kube-bench | Compliance / CIS | Same |
| PLAN → file mapping | Trace plan items to content | `kuberenets/PLAN_IMPLEMENTATION_MAP.md` |
| VPC CNI IP exhaustion | Real EKS outages | `networking_deep_dive.md` (VPC CNI section) |

---

## Recommended gap-fill order (if time is limited)

1. **Gateway API** + **Organizations/SCPs** (if AWS platform role)  
2. **OTel appendix** (already in repo — read it)  
3. **Velero** or **etcd snapshot** lab (pick one DR story)  
4. **SQS/SNS** chapter from AWS docs (half day)  
5. **Cosign** quickstart (supply chain angle)

---

*Generated from a full pass over markdown files in this repository. Update this matrix when you add new guides.*
