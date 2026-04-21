# 05 — Security, RBAC, and policy

## 1. Authentication (who are you?)

Kubernetes supports multiple **authenticators** (often combined):

- X509 client certs
- Bearer tokens (static, service account JWT, OIDC)
- Webhook token authentication
- **Anonymous** requests (can be disabled)

**kubeconfig** holds cluster URL, CA, and credentials (cert or token).

**In interviews:** EKS uses `aws eks get-token`; GKE uses gcloud credentials; many enterprises add **OIDC** (e.g., Okta) to API server `--oidc-*` flags.

---

## 2. Authorization (what may you do?)

After authN, **authorizers** run in order (e.g., Node, RBAC, Webhook). **RBAC** is standard.

### RBAC objects

| Object | Scope |
|--------|--------|
| **Role** | Namespace |
| **ClusterRole** | Cluster-wide |
| **RoleBinding** | Bind subject to Role in namespace |
| **ClusterRoleBinding** | Bind subject to ClusterRole cluster-wide |

**Subjects:** `User`, `Group`, `ServiceAccount`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: ci-bot
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**`roleRef` is immutable** — to change permissions, create a new RoleBinding or patch Role.

---

## 3. ServiceAccounts (SA)

Each namespace has `default` SA. Pods get an SA (mount token at `/var/run/secrets/kubernetes.io/serviceaccount` unless `automountServiceAccountToken: false`).

**Bound tokens** (1.22+): time-limited, audience-bound projected volumes preferred over long-lived legacy tokens.

---

## 4. Projected service account token (short-lived)

```yaml
      volumes:
        - name: kube-api-access
          projected:
            sources:
              - serviceAccountToken:
                  path: token
                  expirationSeconds: 3600
                  audience: my-audience
```

---

## 5. Pod Security Standards (PSS)

Replaces deprecated **PodSecurityPolicy** (removed 1.25). Enforced via **namespace labels** (`pod-security.kubernetes.io/enforce: restricted`) or admission (e.g., Kyverno, OPA Gatekeeper).

**Levels:** `privileged`, `baseline`, `restricted` — control host namespaces, capabilities, root user, seccomp, etc.

---

## 6. Admission controllers

**Mutating** (modify object) and **validating** (allow/deny). Examples: `ResourceQuota`, `LimitRanger`, `NamespaceLifecycle`, `ValidatingAdmissionWebhook`.

**Interview:** “What runs after API validation but before etcd?” → **admission chain**.

---

## 7. imagePullSecrets

For private registries, reference a `docker-registry` Secret in Pod spec:

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

---

## 8. Security best-practice checklist

- Least-privilege **RBAC**; avoid `cluster-admin` for apps
- **NetworkPolicy** default deny + explicit allows
- **PSS** `restricted` where possible
- **Secrets** encryption at rest; integrate external secret operator
- Scan images (Trivy, etc.); sign images (cosign)
- Disable anonymous auth; audit logs for API server

---

## Quick recap

- **AuthN** → **AuthZ (RBAC)** → **Admission** → etcd.
- **Role + RoleBinding** for namespace scope; **ClusterRole** for cluster or aggregation.
- **PSS** enforces safe Pod fields; **PSP** is historical.

### Interview prompts

1. Difference between RoleBinding and ClusterRoleBinding with a namespaced Role?
2. How does a Pod call the Kubernetes API securely?
3. What replaced PodSecurityPolicy, and how is it enforced?
4. What is the risk of giving a Deployment’s SA `list secrets` cluster-wide?
