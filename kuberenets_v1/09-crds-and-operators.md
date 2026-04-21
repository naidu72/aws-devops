# 09 — CRDs, custom resources, and operators

## 1. Why CRDs exist

Kubernetes ships a **fixed set of core APIs** (Pod, Deployment, …). **CustomResourceDefinitions (CRDs)** extend the API server so you can store and version **new kinds** (e.g. `KafkaTopic`, `Certificate`, `Application`) in **etcd** with the same patterns as built-in resources: `kubectl`, RBAC, watches, controllers.

---

## 2. CustomResourceDefinition (CRD)

A **CRD** is a cluster-scoped object that **defines**:

| Field / concept | Purpose |
|-----------------|--------|
| **group** | API group, e.g. `stable.example.com` |
| **versions** | One or more served versions (`v1`, `v1alpha1`, …) |
| **scope** | `Namespaced` or `Cluster` |
| **names** | plural, singular, **kind**, shortNames (optional) |
| **schema** | OpenAPI v3 validation (`openAPIV3Schema`) — optional but recommended |

After you apply a CRD, the API server serves:

`apis/<group>/<version>/namespaces/<ns>/<plural>`  
or for cluster-scoped: `apis/<group>/<version>/<plural>`

**Discover:**

```bash
kubectl api-resources | grep -i yourgroup
kubectl get crd
kubectl explain mywidgets.stable.example.com
```

---

## 3. Custom resource (instance)

An instance is a YAML object with `apiVersion: <group>/<version>`, `kind: <Kind>`, `metadata`, and **`spec`** / **`status`** (status often written by a controller, not users).

**Example — minimal CRD + one instance** (illustrative only; validate on a throwaway cluster):

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.stable.example.com
spec:
  group: stable.example.com
  names:
    kind: Widget
    listKind: WidgetList
    plural: widgets
    singular: widget
    shortNames: [wdg]
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
              required: [replicas]
            status:
              type: object
              properties:
                phase:
                  type: string
      subresources:
        status: {}
```

```yaml
apiVersion: stable.example.com/v1
kind: Widget
metadata:
  name: demo
  namespace: default
spec:
  replicas: 3
```

**subresources.status** — enables `/status` subresource so controllers update **status** without fighting user changes to `spec` (RBAC can split `update` vs `update/status`).

---

## 4. Versioning and conversion

- Multiple **served** versions possible; exactly one **`storage: true`** version in etcd.
- **Conversion webhooks** translate between API versions so clients can use `v1beta1` while storage is `v1`.
- **Deprecation:** migrate users and manifests before removing a version from `served: true`.

---

## 5. Validation and defaults

- **Structural schema** rejects invalid `spec` at admission (when schema is strict).
- **CEL validation rules** (Kubernetes 1.25+ on CRDs) — declarative constraints without a webhook.
- **Mutating / validating admission webhooks** — for complex logic; register via `validatingwebhookconfigurations` / `mutatingwebhookconfigurations` targeting the CRD’s API.

---

## 6. Controllers and the operator pattern

A **controller** runs a **reconcile loop**: watch CR (and related objects) → compare **desired** (`spec`) vs **actual** (cluster state) → create/update/delete Pods, Services, etc. → write **`status`**.

**Operator** = controller + CRD(s) + packaging (often Helm/Olm) + upgrade/backup logic — domain-specific automation (databases, queues, ingress, GitOps apps).

Common **frameworks:**

| Tool | Notes |
|------|--------|
| **kubebuilder** | Go controller-runtime; `CRD` + manager scaffold |
| **Operator SDK** | Ansible/Helm/Go; Red Hat ecosystem |
| **Kopf** | Python operators |

**Interview line:** “A CRD alone only adds **schema** to the API; something must **act** on it — usually a controller in a Deployment.”

---

## 7. RBAC for custom resources

Rules use `apiGroups: [stable.example.com]`, `resources: [widgets]`, `verbs: [get, list, watch, create, update, patch, delete]`.  
For status subresource: `resources: [widgets/status]`.

---

## 8. Aggregated API servers (contrast)

**Extension API servers** register extra APIs via **APIService** and can use their own storage — heavier than CRDs. CRDs are the **default** way teams extend Kubernetes without running a separate API process.

---

## 9. Troubleshooting CRDs

| Symptom | Check |
|---------|--------|
| `kubectl apply` rejected | `kubectl describe crd …`; schema / required fields; server version supports CRD features |
| Controller not reconciling | Controller logs, RBAC, leader election, correct **GVK** watched |
| Two controllers fight | Owner references, finalizers, single reconciler for one CR kind |
| Version mismatch | `kubectl get crd -o yaml` — served vs storage versions |

**Finalizers** — metadata.finalizers block deletion until controller removes them (common in operators).

---

## 10. Pointers (deeper reading in-repo)

Sibling pack: [`../kuberenets/crds_operators_guide.md`](../kuberenets/crds_operators_guide.md) (if present) for longer-form notes.

---

## Quick recap

- **CRD** defines a new **Kind** and **schema**; instances are **custom resources** stored in etcd.
- **Operators** = CRD + **controller** (+ often webhooks) to reconcile real cluster state.
- Use **`subresources.status`**, **RBAC** on resources and `*/status`, and **versioning/conversion** for production APIs.

### Interview prompts

1. What is the difference between a CRD and a custom resource instance?
2. Why would you enable the **status** subresource on a CRD?
3. What runs the reconcile loop for a CRD—Kubernetes itself or your code?
4. How would you restrict who can create `Widget` objects in namespace `team-a`?
5. When might you prefer an **aggregated API server** over a CRD?
