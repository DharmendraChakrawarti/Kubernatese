# Module 08: Storage & Volumes

> **Level:** 🟡 Intermediate  
> **Objective:** Understand Kubernetes storage — Volumes, PV, PVC, StorageClasses.

---

## 8.1 The Storage Problem

Containers are **ephemeral** — data dies when the container restarts. Kubernetes provides a layered storage system:

```
StorageClass → PersistentVolume (PV) → PersistentVolumeClaim (PVC) → Pod Volume Mount
```

---

## 8.2 Volume Types

### Ephemeral Volumes

```yaml
# emptyDir — temporary, shared between containers, deleted when Pod dies
volumes:
- name: cache
  emptyDir: {}            # Uses node's filesystem
- name: fast-cache
  emptyDir:
    medium: Memory         # Uses RAM (tmpfs)
    sizeLimit: 256Mi
```

### ConfigMap / Secret as Volumes

```yaml
volumes:
- name: config
  configMap:
    name: app-config
- name: secrets
  secret:
    secretName: app-secrets
    defaultMode: 0400       # Read-only
```

### hostPath (Node's Filesystem)

```yaml
volumes:
- name: host-data
  hostPath:
    path: /var/log
    type: Directory         # Directory | File | DirectoryOrCreate
```

> ⚠️ **Avoid in production** — ties Pod to a specific node.

---

## 8.3 PersistentVolume (PV)

**Cluster-wide storage resource** provisioned by an admin or dynamically.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:                    # For testing only
    path: /mnt/data
---
# NFS example
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteMany
  nfs:
    server: 192.168.1.100
    path: /exports/data
```

### Access Modes

| Mode | Abbreviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | **RWO** | One node can mount read-write |
| ReadOnlyMany | **ROX** | Many nodes can mount read-only |
| ReadWriteMany | **RWX** | Many nodes can mount read-write |
| ReadWriteOncePod | **RWOP** | One pod can mount read-write |

### Reclaim Policies

| Policy | Behavior |
|--------|----------|
| **Retain** | PV preserved after PVC deletion (manual cleanup) |
| **Delete** | PV and underlying storage deleted |
| **Recycle** | ⚠️ Deprecated — basic `rm -rf` |

---

## 8.4 PersistentVolumeClaim (PVC)

**User's request for storage.** Binds to a matching PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard    # Must match PV or StorageClass
```

### Using PVC in a Pod

```yaml
spec:
  containers:
  - name: app
    image: my-app:v1
    volumeMounts:
    - name: data
      mountPath: /app/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

### PV-PVC Binding

```
PVC Request              PV Available          Match?
──────────────           ──────────────        ──────
5Gi RWO standard    ←→   10Gi RWO standard    ✅ (capacity ≥ request)
5Gi RWX standard    ←→   10Gi RWO standard    ❌ (access mode mismatch)
```

### PVC Status

| Status | Meaning |
|--------|---------|
| **Pending** | No matching PV found |
| **Bound** | Successfully bound to a PV |
| **Lost** | PV was deleted while PVC existed |

---

## 8.5 StorageClass (Dynamic Provisioning)

**Automatically creates PVs** when a PVC is created.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs   # Cloud-specific
parameters:
  type: gp3
  iopsPerGB: "3000"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### Common Provisioners

| Cloud | Provisioner | Storage Type |
|-------|------------|-------------|
| AWS | `ebs.csi.aws.com` | EBS volumes |
| GCP | `pd.csi.storage.gke.io` | Persistent Disks |
| Azure | `disk.csi.azure.com` | Azure Disks |
| On-prem | `nfs.csi.k8s.io` | NFS shares |

### Volume Binding Modes

| Mode | Behavior |
|------|----------|
| **Immediate** | PV created immediately when PVC is created |
| **WaitForFirstConsumer** | PV created only when Pod is scheduled (better for topology) |

---

## 8.6 Volume Expansion

```yaml
# StorageClass must have allowVolumeExpansion: true
# Then simply edit the PVC:
kubectl edit pvc my-pvc
# Change storage from 5Gi to 10Gi
```

---

## 8.7 StatefulSet Volume Claim Templates

```yaml
# Each replica gets its own PVC
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: fast-ssd
    resources:
      requests:
        storage: 10Gi
# Creates: data-app-0, data-app-1, data-app-2
```

---

## 8.8 CSI (Container Storage Interface)

Modern, plugin-based storage architecture:

```
Pod → kubelet → CSI Driver → Cloud API → Disk
```

Common CSI drivers: AWS EBS CSI, GCP PD CSI, Azure Disk CSI, Rook-Ceph, Longhorn, OpenEBS.

---

**Previous:** [← Module 07](./07-services-networking.md) | **Next:** [Module 09 →](./09-configmaps-secrets.md)
