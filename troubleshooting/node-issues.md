# 4. Node Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 4.1 Node in NotReady State

### Symptom
```bash
kubectl get nodes
# NAME           STATUS     ROLES           AGE   VERSION
# worker-1       NotReady   <none>          30d   v1.28.2
```

### Cause
- Kubelet service crashed or stopped on the node.
- Node ran out of disk space, memory, or PIDs.
- Network connectivity lost between the node and the control plane.
- Container runtime (containerd/Docker) crashed.
- Node clock is significantly out of sync (certificate validation fails).

### Diagnosis

```bash
# Step 1: Check node conditions
kubectl describe node worker-1 | grep -A 20 "Conditions:"
# Look for:
#   Ready            False    KubeletNotReady    container runtime not running
#   MemoryPressure   True
#   DiskPressure     True

# Step 2: SSH into the node
ssh user@worker-1

# Step 3: Check kubelet status
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago" | tail -50

# Step 4: Check container runtime
systemctl status containerd
# or: systemctl status docker

# Step 5: Check disk space
df -h
# Look for root or container partitions at 100%

# Step 6: Check memory
free -h

# Step 7: Check if the node can reach the API server
curl -k https://<api-server-ip>:6443/healthz
```

### Resolution

```bash
# Fix 1: Restart kubelet
sudo systemctl restart kubelet

# Fix 2: Restart container runtime
sudo systemctl restart containerd

# Fix 3: Free disk space
sudo crictl rmi --prune                     # remove unused container images
sudo journalctl --vacuum-size=500M          # trim journal logs
sudo rm -rf /var/log/*.gz                   # remove old log archives

# Fix 4: If node is completely unrecoverable
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
kubectl delete node worker-1
# Then re-join the node:
# kubeadm join <api-server>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 4.2 Node Disk Pressure / Memory Pressure / PID Pressure

### Symptom
```bash
kubectl describe node worker-1 | grep -i pressure
# DiskPressure     True
# MemoryPressure   True
# PIDPressure      True
```
Pods start getting **evicted** from the node.

### Cause
- **DiskPressure:** Disk usage exceeds eviction threshold (default: ~85% for `nodefs`, ~100% for `imagefs`).
- **MemoryPressure:** Available memory drops below 100Mi (default threshold).
- **PIDPressure:** Number of running processes exceeds the kubelet's `--pid-limit` (default: ~262144).

### Diagnosis

```bash
# SSH into the node

# Disk
df -h
du -sh /var/lib/containerd/*    # container images and layers
du -sh /var/log/*               # log files

# Memory
free -h
top -bn1 | head -20

# PIDs
ps aux | wc -l
cat /proc/sys/kernel/pid_max
```

### Resolution

**Kubelet eviction thresholds – customize if defaults are too aggressive:**

```bash
# File: /var/lib/kubelet/config.yaml (on the node)
```
```yaml
evictionHard:
  memory.available: "200Mi"          # ✅ evict pods when free memory < 200Mi
  nodefs.available: "10%"            # ✅ evict when disk < 10% free
  imagefs.available: "15%"           # ✅ evict when image disk < 15% free
  pid.available: "5%"               # ✅ evict when PIDs < 5% available
evictionSoft:
  memory.available: "500Mi"          # ✅ soft threshold – starts evicting gently
  nodefs.available: "15%"
evictionSoftGracePeriod:
  memory.available: "1m30s"          # ✅ wait 1.5 min before soft eviction
  nodefs.available: "2m"
```

After editing, restart kubelet:
```bash
sudo systemctl restart kubelet
```

**Clean up disk space:**
```bash
# Remove unused images
sudo crictl rmi --prune

# Garbage collect containers
sudo crictl rm $(sudo crictl ps -a -q --state exited)

# Clear old journal logs
sudo journalctl --vacuum-time=3d
```

---

## 4.3 Taint Preventing Pod Scheduling

### Symptom
Pods are `Pending` and events say `0/3 nodes are available: 3 node(s) had taints that the pod didn't tolerate`.

### Diagnosis

```bash
# Check taints on all nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Common taints:
# node-role.kubernetes.io/control-plane:NoSchedule  → control plane node
# node.kubernetes.io/not-ready:NoSchedule           → node not ready
# node.kubernetes.io/disk-pressure:NoSchedule       → disk full
# custom-key=custom-value:NoSchedule                → user-defined taint
```

### Resolution

#### Option A: Add a toleration to the pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      tolerations:                                   # ✅ ADD THIS BLOCK
        - key: "dedicated"                           # ✅ matches the taint key
          operator: "Equal"                          # ✅
          value: "gpu-workloads"                     # ✅ matches the taint value
          effect: "NoSchedule"                       # ✅ matches the taint effect
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
```

#### Option B: Remove the taint from the node

```bash
# Remove a specific taint
kubectl taint nodes worker-1 dedicated=gpu-workloads:NoSchedule-
#                                                              ^ the trailing dash removes it

# Verify taint is removed
kubectl describe node worker-1 | grep Taints
# Taints: <none>
```

#### Option C: Schedule on control plane nodes (for single-node clusters)

```yaml
      tolerations:
        - key: "node-role.kubernetes.io/control-plane"
          operator: "Exists"
          effect: "NoSchedule"
```

Or remove the taint permanently:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## 4.4 Draining a Node for Maintenance

### Scenario
You need to take a node offline for OS updates, hardware changes, etc.

### Procedure

```bash
# Step 1: Mark node as unschedulable (no new pods)
kubectl cordon worker-1
# node/worker-1 cordoned

# Step 2: Evict all pods gracefully
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=60

# Step 3: Perform maintenance on the node
ssh user@worker-1
sudo apt update && sudo apt upgrade -y
sudo reboot

# Step 4: Uncordon the node (allow pods again)
kubectl uncordon worker-1
# node/worker-1 uncordoned
```

**If drain is stuck** (PodDisruptionBudget prevents eviction):
```bash
# Check what PDB is blocking
kubectl get pdb -A

# Example PDB:
# NAME       MIN AVAILABLE   MAX UNAVAILABLE   CURRENT   DESIRED
# my-pdb     2               N/A               2         3
# ← can't evict because it would drop below minAvailable

# Temporarily lower minAvailable or increase replicas
kubectl patch pdb my-pdb -p '{"spec":{"minAvailable":1}}'
```

---

## 4.5 Node Kernel or OS Issues

### Symptom
- `dmesg` shows kernel panics, filesystem errors, or hardware failures.
- Node sporadically becomes `NotReady` and recovers.

### Diagnosis

```bash
# SSH into the node

# Check kernel messages
dmesg | tail -50
dmesg | grep -i -E "error|fail|oom|panic"

# Check system logs
journalctl -p err --since "1 hour ago"

# Check filesystem
sudo fsck -n /dev/sda1    # dry-run, non-destructive

# Check for hardware errors
sudo smartctl -a /dev/sda
```

### Resolution
- Apply kernel updates: `sudo apt update && sudo apt upgrade linux-image-*`
- If hardware is failing, drain the node and replace it.
- For filesystem corruption: drain the node, unmount, run `fsck`, remount.

---

*[← Back to Index](./notes.md)*
