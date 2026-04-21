# 04 — Storage and configuration

## 1. Volumes in Pods

**Ephemeral** volume types: `emptyDir` (disk or RAM `medium: Memory`), lifecycle tied to Pod.

**persistent** types: reference **PersistentVolumeClaim** (PVC).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-emptydir
spec:
  volumes:
    - name: cache
      emptyDir: {}
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: cache
          mountPath: /cache
```

---

## 2. PersistentVolume (PV) and PersistentVolumeClaim (PVC)

- **PV:** cluster-scoped storage resource (NFS, EBS volume, GCE PD, CSI driver volume, etc.).
- **PVC:** namespaced **request** for storage (size, access mode, optionally StorageClass).

**Binding:** control plane matches PVC to suitable PV (static) or **dynamic provisioning** creates PV via **StorageClass**.

### Access modes

| Mode | Meaning |
|------|---------|
| **ReadWriteOnce (RWO)** | Single node read/write |
| **ReadOnlyMany (ROX)** | Many nodes read-only |
| **ReadWriteMany (RWX)** | Many nodes read/write (NFS, EFS, some CSI) |

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

---

## 3. StorageClass

Defines **provisioner** (e.g., `ebs.csi.aws.com`), **parameters** (type, iops), **reclaimPolicy** (`Delete` vs `Retain`), **volumeBindingMode** (`Immediate` vs `WaitForFirstConsumer` — latter co-locates Pod and volume for topology-aware provisioning).

---

## 4. CSI (Container Storage Interface)

Out-of-tree drivers implement **provision/attach/mount**. Interview points: CSI sidecars (`csi-provisioner`, `node-driver-registrar`, etc.), **VolumeSnapshot** API for backups.

---

## 5. ConfigMap

Non-sensitive config as key-value or file content; mounted as env, volume, or args.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    log.level=INFO
```

```yaml
      envFrom:
        - configMapRef:
            name: app-config
```

**Note:** ConfigMaps are **not** secret; RBAC still applies; large objects have size limits (~1Mi default etcd practical limit).

---

## 6. Secret

Opaque, `docker-registry`, `tls`, or `basic-auth` types. Stored **base64-encoded** in etcd — **not encryption by default**; enable **encryption at rest** on API server for production.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
stringData:
  username: appuser
  password: changeme
```

Mount as volume (files) or `env` (less ideal — visible in `/proc`).

---

## 7. Downward API

Expose Pod/namespace metadata to containers:

```yaml
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

---

## Quick recap

- **PVC** requests storage; **StorageClass** + CSI **dynamically provisions** PVs.
- Access modes and **volumeBindingMode** matter for scheduling and HA.
- **ConfigMap/Secret** for config; protect Secrets with RBAC + etcd encryption + external secret managers (Vault, ESO, etc.).

### Interview prompts

1. Static vs dynamic provisioning — walk through the objects involved.
2. Why use `WaitForFirstConsumer` on a StorageClass?
3. Why should you avoid putting Secrets only in environment variables?
4. What happens to a PVC when you delete a StatefulSet’s `volumeClaimTemplates` Pods?
