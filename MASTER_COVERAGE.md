# DevOps & Kubernetes interview prep — **one-stop coverage**

**Repository root:** `/home/frontier/devops-interview-prep/aws-devops`  
**Purpose:** One file that maps *everything* in this tree, suggests a study order, and captures **lab-hardened** notes (Istio ingress, Gateway conflicts, VirtualService ordering) so you do not hunt across folders.

> **Bookmark this file.** All paths below are relative to this directory unless noted.

---

## 0. Are any important concepts missing?

**Yes — a few typical senior-level topics are only summarized or absent.** See the full audit:

| Document | Purpose |
|----------|---------|
| **[CONCEPT_COVERAGE_MATRIX.md](CONCEPT_COVERAGE_MATRIX.md)** | Topic × depth (Deep / Summary / Mention / **Gap**) + where to read in-repo + short external pointers for **gaps** (Gateway API, Organizations/SCPs, MSK, Terragrunt, Kyverno, cosign, Velero, Linkerd, RUM, etc.) |

Use the matrix when you feel “something’s missing”—it maps **every major area** to a file **or** marks it as a gap with a fix path.

---

## 1. How to use this repo (two study tracks)

| Track | Audience | Path |
|--------|----------|------|
| **A — AWS / broad DevOps** | Platform, AWS services, phases, mock interviews | [§2 Program phases](#2-program-phases-root-markdown-files) → [§4 Meta & quick refs](#4-meta--quick-reference) |
| **B — Deep Kubernetes** | CKA-style depth, EKS, networking, operators | [§3 Kubernetes deep dive (`kuberenets/`)](#3-kubernetes-deep-dive-kuberenets) → `kuberenets/INDEX_COMPLETE_STUDY_PACKAGE.md` |

**Suggested first week (combined):** Days 1–2 Phase 1 + `kuberenets/core_resources_*`; Days 3–4 Phase 2–3 + `kuberenets/networking_deep_dive.md` + `kuberenets/eks_architecture_guide.md`; Days 5–7 Phase 4–5 + `kuberenets/MONITORING_SCANNING_CLOUD_TOOLS.md` + mock sections.

---

## 2. Program phases (root Markdown files)

| Order | File | What it covers |
|-------|------|----------------|
| Entry | [README-INTERVIEW-PREP.md](README-INTERVIEW-PREP.md) | Full program narrative, phase list, schedules |
| Overview | [EXECUTION-SUMMARY.md](EXECUTION-SUMMARY.md) | What exists, stats, how pieces fit |
| Cheat sheet | [QUICK-REFERENCE.md](QUICK-REFERENCE.md) | Top questions, commands, interview tips |
| Phase 1 theory | [Phase1-AWS-Fundamentals.md](Phase1-AWS-Fundamentals.md) | IAM, EC2, S3, RDS, VPC, Route53, etc. |
| Phase 1 labs | [Phase1-Hands-On-Labs.md](Phase1-Hands-On-Labs.md) | AWS CLI hands-on labs |
| Phase 2 | [Phase2-Containerization-Orchestration.md](Phase2-Containerization-Orchestration.md) | Docker, ECS, Kubernetes fundamentals |
| Phase 3 | [Phase3-CICD-Automation.md](Phase3-CICD-Automation.md) | CI/CD, pipelines, IaC touchpoints |
| Phase 4 | [Phase4-Advanced-Topics.md](Phase4-Advanced-Topics.md) | Monitoring, security, DR, cost |
| Phase 5 | [Phase5-Interview-Practice.md](Phase5-Interview-Practice.md) | Architectures, troubleshooting, mock interview |

**Onboarding text file:** [START-HERE.txt](START-HERE.txt) — quick list of phase files (keep in sync mentally with this section).

---

## 3. Kubernetes deep dive (`kuberenets/`)

**Folder:** `kuberenets/` (spelling preserved in tree).

### 3.1 Start-here inside Kubernetes pack

| File | Role |
|------|------|
| [kuberenets/README.md](kuberenets/README.md) | File index (same info as tables below, K8s-focused) |
| [kuberenets/INDEX_COMPLETE_STUDY_PACKAGE.md](kuberenets/INDEX_COMPLETE_STUDY_PACKAGE.md) | Roadmap, topic checklist |
| [kuberenets/study_summary.md](kuberenets/study_summary.md) | Short module + file inventory; points **here** for full-repo coverage |
| [kuberenets/PLAN_IMPLEMENTATION_MAP.md](kuberenets/PLAN_IMPLEMENTATION_MAP.md) | Plan todo IDs → file/section mapping |
| [kuberenets/k8s_study_guide.md](kuberenets/k8s_study_guide.md) | Structured timeline |

### 3.2 Topic → file map (all under `kuberenets/`)

| Topic | Files |
|--------|--------|
| Core workloads & Services | `core_resources_interview_qa.md`, `core_resources_guide.yaml`, `core_resources_guide.md` |
| Storage | `storage_concepts_guide.md` |
| Networking | `networking_deep_dive.md` |
| CRDs / Operators | `crds_operators_guide.md` |
| EKS | `eks_architecture_guide.md` |
| AKS | `aks_architecture_guide.md` |
| Bare metal | `bare_metal_kubernetes_guide.md` |
| Advanced umbrella | `remaining_modules_summary.md` |
| Observability & tools | `MONITORING_SCANNING_CLOUD_TOOLS.md` |
| CI/CD tools | `CICD_DEPLOYMENT_TOOLS.md` |
| Completion checklists | `INTERVIEW_PREP_COMPLETE_FINAL.md`, `INTERVIEW_PREP_COMPLETE.md` |

### 3.3 Historical fixes (so you do not chase ghosts)

- Older drafts referenced `/tmp/...`; **all current paths are under `kuberenets/`** or this repo root.
- Networking file name is **`networking_deep_dive.md`** (not `networking_guide.md`).

---

## 4. Meta & quick reference

| Need | File |
|------|------|
| Interview readiness checklist | `kuberenets/INTERVIEW_PREP_COMPLETE_FINAL.md` |
| Print-friendly Q&A / commands | `QUICK-REFERENCE.md` |
| “What shipped / stats” | `EXECUTION-SUMMARY.md` |

---

## 5. Embedded lab notes — Istio, ingress, Gateways (single reference)

*These are the recurring issues from hands-on work; they belong in “one place” so you do not re-debug from memory.*

### 5.1 Find your real ingress (not only `istio-system`)

Many clusters use a **Helm “gateway”** install:

- Namespace often **`istio-ingress`**, Deployment/Service often **`istio-ingress`**.
- Label for `Gateway.spec.selector` is often **`istio: ingress`**, not `istio: ingressgateway`.

```bash
kubectl get deploy -A -l 'istio in (ingress,ingressgateway)'
kubectl get svc -A | grep -i ingress
# Port-forward the Service that backs your ingress, e.g.:
# kubectl port-forward -n istio-ingress svc/istio-ingress 8080:80
```

### 5.2 `404` through ingress (Envoy)

Usually **no route (NR)** — not your app returning HTTP 404.

- Put **`Gateway`**, **`VirtualService`**, and **workload Services** in the **same logical setup** (same namespace unless you intentionally qualify names).
- Port-forward the **ingress** Service, not the app Service.
- Verify: `kubectl get gateway,virtualservice -n <ns>`.

### 5.3 `istioctl analyze` — **IST0145** (Gateway conflict)

Two `Gateway` objects attach to the **same** ingress workload, **same port**, overlapping **`hosts`** (e.g. both `*`) → conflict.

**Mitigations:** use a **dedicated host** on one Gateway + matching `VirtualService.spec.hosts` and `curl -H "Host: …"`; or narrow/remove the duplicate Gateway; or use another port.

### 5.4 VirtualService rule order — `/api` vs `/`

For `fetch('/api')` via the **same host** as the UI (through ingress):

- Put **`/api` → backend** *above* **`/` → frontend** in the same `VirtualService`.
- Otherwise all traffic may hit nginx first; mesh-only routing is clearer at the edge.

### 5.5 Analyze noise — IST0103 / IST0118

- **IST0103:** sidecar missing → enable injection for the namespace + **rollout restart**.
- **IST0118:** name Service ports (`http`, `http-web`, …) for Istio protocol detection.

---

## 6. Optional material outside this repo clone

If you keep a larger “interview_coverage” or home folder, these often exist **next to** this tree (not always committed here):

| Area | Typical location | Contents |
|------|------------------|----------|
| EKS + Istio labs | `../eks-kubernetes/` (sibling of `aws-devops`) | `06-hands-on-labs.md`, `samples/httpbin.yaml`, Istio Gateway samples |
| Terraform / CloudFormation | `../terraform-iac/` | Terraform fundamentals, CFN comparison |
| Personal Istio demo | `~/istio-demo/` or similar | `gateway.yaml`, `virtualservice.yaml`, frontend/backend split |

**Rule:** If a sibling path is missing, clone or copy your full prep bundle — **this file still lists the conceptual coverage** in §5.

---

## 7. Single-line “coverage checklist”

Use this to confirm breadth before an interview:

- [ ] AWS core: Phase 1 + Phase 1 labs  
- [ ] Containers & orchestration: Phase 2 + `kuberenets/core_resources_*` + `networking_deep_dive.md`  
- [ ] CI/CD & GitOps: Phase 3 + `kuberenets/CICD_DEPLOYMENT_TOOLS.md`  
- [ ] EKS (or target cloud): `kuberenets/eks_architecture_guide.md`  
- [ ] Observability & mesh: `kuberenets/MONITORING_SCANNING_CLOUD_TOOLS.md` + §5 above  
- [ ] Mock / scenarios: Phase 5 + `INTERVIEW_PREP_COMPLETE_FINAL.md`  
- [ ] Quick review: `QUICK-REFERENCE.md`  
- [ ] **Gap pass:** [CONCEPT_COVERAGE_MATRIX.md](CONCEPT_COVERAGE_MATRIX.md) (mark **Gap** rows you care about)

---

## 8. File count snapshot (this repository)

| Location | Markdown files (approx.) |
|----------|---------------------------|
| Repo root | 12 phase/meta files (includes `MASTER_COVERAGE.md`, `CONCEPT_COVERAGE_MATRIX.md`) |
| `kuberenets/` | 19 files |
| **Total** | **~29** `.md` files tracked in this tree |

*(If you add `eks-kubernetes/` or `terraform-iac/` later, bump the count and optionally extend §6.)*

---

*Canonical index: this document. Last updated: 2026-04-21.*
