# 3. Storage & Volume Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 3.1 PVC Stuck in Pending

### Symptom
```bash
kubectl get pvc
# NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# data-pvc   Pending                                       standard       10m
```

### Cause
- No PersistentVolume (PV) matches the PVC's `storageClassName`, `accessModes`, or size.
- StorageClass does not exist or doesn't have a provisioner.
- Dynamic provisioner quota exceeded (cloud disk limits).
- StorageClass `volumeBindingMode: WaitForFirstConsumer` — PVC stays Pending until a pod uses it.

### Diagnosis

```bash
# Check PVC events
kubectl describe pvc data-pvc
# Events:
#   Warning  ProvisioningFailed  no persistent volumes available for this claim

# Check available PVs
kubectl get pv

# Check StorageClasses
kubectl get storageclass
```

### Resolution

#### Case A: No StorageClass – create one (for cloud)

```yaml
# File: storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd       # GKE
# provisioner: ebs.csi.aws.com          # EKS
# provisioner: disk.csi.azure.com       # AKS
parameters:
  type: pd-ssd                          # GKE SSD
  # type: gp3                           # AWS gp3
  # skuName: Premium_LRS                # Azure Premium SSD
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer  # ✅ bind only when pod is scheduled
allowVolumeExpansion: true               # ✅ allow resizing PVCs later
```

#### Case B: PVC and PV for static provisioning

**PV (created by admin):**
```yaml
# File: pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-pv
spec:
  capacity:
    storage: 10Gi                    # ✅ must be >= PVC request
  accessModes:
    - ReadWriteOnce                  # ✅ must match PVC
  storageClassName: manual           # ✅ must match PVC
  hostPath:                          # for single-node testing only
    path: /mnt/data
```

**PVC:**
```yaml
# File: pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce                  # ✅ must match PV
  storageClassName: manual           # ✅ must match PV
  resources:
    requests:
      storage: 10Gi                  # ✅ must be <= PV capacity
```

**Deployment using the PVC:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          volumeMounts:
            - name: data-volume
              mountPath: /app/data        # ✅ where the app reads/writes data
      volumes:
        - name: data-volume
          persistentVolumeClaim:
            claimName: data-pvc           # ✅ must match PVC name
```

---

## 3.2 Volume Mount Permission Denied

### Symptom
Container logs show:
```
Error: EACCES: permission denied, open '/app/data/file.txt'
```

### Cause
- Volume is owned by `root` (UID 0) but the container runs as a non-root user.
- `fsGroup` is not set in the pod's security context.
- NFS volume with `root_squash` enabled.

### Resolution

**❌ Broken – non-root user can't write to volume:**
```yaml
    spec:
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          securityContext:
            runAsUser: 1000              # ⚠️ runs as UID 1000
          volumeMounts:
            - name: data-volume
              mountPath: /app/data       # ⚠️ owned by root, UID 1000 can't write
      volumes:
        - name: data-volume
          persistentVolumeClaim:
            claimName: data-pvc
```

**✅ Fixed – set fsGroup to change volume ownership:**
```yaml
    spec:
      securityContext:                   # ✅ POD-level security context
        runAsUser: 1000                  # ✅ run as UID 1000
        runAsGroup: 1000                 # ✅ primary group GID 1000
        fsGroup: 1000                    # ✅ all mounted volumes owned by GID 1000
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          volumeMounts:
            - name: data-volume
              mountPath: /app/data       # ✅ now writable by UID/GID 1000
      volumes:
        - name: data-volume
          persistentVolumeClaim:
            claimName: data-pvc
```

**Alternative – use an initContainer to fix permissions:**
```yaml
    spec:
      initContainers:
        - name: fix-permissions
          image: busybox:1.36
          command: ["sh", "-c", "chown -R 1000:1000 /app/data"]
          volumeMounts:
            - name: data-volume
              mountPath: /app/data
          securityContext:
            runAsUser: 0                 # ✅ init runs as root to chown
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          securityContext:
            runAsUser: 1000
          volumeMounts:
            - name: data-volume
              mountPath: /app/data
```

---

## 3.3 Multi-Attach Error for Volumes

### Symptom
```
Warning  FailedAttachVolume  Multi-Attach error for volume "pvc-abc123":
         Volume is already exclusively attached to one node
```

### Cause
- PV uses `ReadWriteOnce` (RWO) but multiple pods on **different nodes** try to mount it.
- RWO means the volume can only be attached to **one node** at a time.

### Resolution

#### Option A: Use ReadWriteMany (RWX) storage

```yaml
# PVC with RWX access mode — requires NFS, EFS, or similar storage backend
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data-pvc
spec:
  accessModes:
    - ReadWriteMany              # ✅ multiple nodes can mount simultaneously
  storageClassName: efs-sc       # ✅ use a StorageClass that supports RWX
  resources:
    requests:
      storage: 10Gi
```

#### Option B: Force pods to the same node with pod affinity

```yaml
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - my-app
              topologyKey: kubernetes.io/hostname    # ✅ same node
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
```

#### Option C: Use a single replica
If the workload doesn't need horizontal scaling:
```yaml
spec:
  replicas: 1                    # ✅ only one pod, one node, no multi-attach
  strategy:
    type: Recreate               # ✅ kills old pod before creating new one
```

---

## 3.4 PVC Resize / Expand Volume

### Symptom
Application is running out of disk space and you need to increase the PVC size.

### Resolution

**Step 1: Ensure the StorageClass allows expansion:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-sc
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true        # ✅ MUST be true
```

**Step 2: Patch the PVC to a larger size:**
```bash
kubectl patch pvc data-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

**Step 3: For filesystem-based volumes, you may need to restart the pod** so the filesystem is resized:
```bash
kubectl rollout restart deployment/my-app
```

> **⚠️ Note:** You can only **increase** PVC size. Shrinking is not supported.

---

*[← Back to Index](./notes.md)*
