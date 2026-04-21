# 03 — Networking and Services

## 1. Kubernetes networking requirements

- Pods can reach each other **without NAT** (same or different nodes — CNI-dependent).
- Agents on a node can reach all Pods on that node.
- Path from host network to Pod IP works (for kubelet, debugging).

**CNI** plugins implement this (routing, overlay, policy).

---

## 2. Service resource

A **Service** is a stable virtual IP (**ClusterIP**) and DNS name that load-balances to **Endpoints** (or **EndpointSlices**) — the set of healthy Pod IPs matching `selector`.

### Service types

| Type | Behavior |
|------|----------|
| **ClusterIP** | Internal VIP only (default) |
| **NodePort** | Exposes port on each node’s IP (high port range) |
| **LoadBalancer** | Requests cloud LB (AWS ELB, GCP LB, etc.) — needs CCM |
| **ExternalName** | CNAME DNS to external name (no proxy) |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

**Headless Service:** `clusterIP: None` — DNS returns **Pod A records** directly (`StatefulSet` use case).

---

## 3. Endpoints and EndpointSlice

- Legacy **Endpoints** object; modern clusters use **EndpointSlice** (scalable sharding).
- **kube-proxy** (or eBPF data plane) watches these and programs forwarding rules.

---

## 4. kube-proxy modes (high level)

| Mode | Notes |
|------|--------|
| **iptables** | Rule chains per Service |
| **ipvs** | LVS for better perf at scale |
| **nftables** | Newer backend (Kubernetes 1.29+ trajectory) |

Some CNIs (e.g., Cilium) bypass kube-proxy for Service load balancing.

---

## 5. Cluster DNS

Typically **CoreDNS**. Records:

- `my-svc.my-ns.svc.cluster.local` → ClusterIP (or Pods for headless)
- `my-pod.my-ns.pod.cluster.local` — optional Pod DNS

**Short names:** same namespace `web` → `web`; cross-namespace `web.otherns`.

---

## 6. Ingress and IngressClass

**Ingress** exposes HTTP(S) routes **to** Services (L7). Requires an **Ingress controller** (nginx, traefik, ALB, etc.).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
spec:
  ingressClassName: nginx
  rules:
    - host: demo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

**IngressClass** selects which controller implementation owns the object.

---

## 7. Gateway API (brief)

Successor to Ingress: **Gateway**, **HTTPRoute**, **GRPCRoute** — more expressive, role-oriented (infrastructure vs app teams). Mention in interviews as “Ingress v2 / standardization.”

---

## 8. NetworkPolicy

**NetworkPolicy** is **default allow-all** unless you use a CNI that enforces policies **and** you create rules — then **default deny** behavior depends on plugin (Calico/Cilium document this clearly).

**Example — allow ingress only from frontend namespace:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
      ports:
        - protocol: TCP
          port: 8080
```

---

## 9. Troubleshooting connectivity (checklist)

1. **Pod running?** `kubectl get pods -o wide`
2. **Service endpoints?** `kubectl get endpoints web`
3. **DNS from Pod?** `kubectl run -it --rm debug --image=busybox:1.36 --restart=Never -- nslookup web`
4. **Policy blocking?** Review NetworkPolicy; CNI must support it.
5. **kube-proxy / CNI** healthy on node?

---

## Quick recap

- **Service** = stable IP/DNS → **Endpoints** → Pod IPs; kube-proxy/CNI implements forwarding.
- **Ingress** = L7 routing to Services; needs a controller.
- **NetworkPolicy** = L4/L3 firewall for Pods (CNI-dependent).

### Interview prompts

1. Explain ClusterIP vs NodePort vs LoadBalancer; when does CCM get involved?
2. What is a headless Service and why use it?
3. How would you debug “Service works by IP but not by DNS”?
4. Difference between Ingress and Gateway API at a high level?
