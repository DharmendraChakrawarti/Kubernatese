# 8. Cluster-Level Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 8.1 etcd Unhealthy / Slow

### Symptom
- `kubectl` commands are extremely slow (10+ seconds).
- etcd health check fails.
- API server logs show: `etcdserver: request timed out`.

### Cause
- Disk I/O too slow — etcd needs SSD with <10ms fsync latency.
- etcd cluster lost quorum (majority of members are down).
- etcd database too large (default limit: 2 GiB).
- Network latency between etcd members too high.

### Diagnosis

```bash
# Check etcd health
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check database size and leader
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table

# Check disk performance
sudo fio --name=etcd-test --rw=write --bs=4k --size=256M \
  --fdatasync=1 --directory=/var/lib/etcd --runtime=30
# 99th percentile fsync should be < 10ms
```

### Resolution

#### Compact and defrag etcd

```bash
# Get the current revision
rev=$(ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=json | jq '.[0].Status.header.revision')

# Compact old revisions
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  compact $rev

# Defrag to reclaim space
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag
```

#### Tune etcd performance

```yaml
# File: /etc/kubernetes/manifests/etcd.yaml (static pod manifest)
# Add/modify these args:
spec:
  containers:
    - name: etcd
      command:
        - etcd
        - --heartbeat-interval=250         # ✅ default: 100ms, increase for high-latency networks
        - --election-timeout=2500          # ✅ default: 1000ms, 10× heartbeat
        - --snapshot-count=5000            # ✅ default: 100000, more frequent snapshots
        - --quota-backend-bytes=8589934592 # ✅ increase db size limit to 8 GiB
        - --auto-compaction-mode=periodic  # ✅ auto compact
        - --auto-compaction-retention=1    # ✅ retain 1 hour of history
```

---

## 8.2 API Server Unavailable

### Symptom
```bash
kubectl get pods
# The connection to the server 10.0.0.5:6443 was refused - did you specify the right host or port?
# OR
# Unable to connect to the server: dial tcp 10.0.0.5:6443: i/o timeout
```

### Cause
- `kube-apiserver` pod/process crashed.
- etcd is down (API server depends on it).
- Certificates expired.
- Load balancer in front of API server is misconfigured or down.
- Node hosting control plane is unreachable.

### Diagnosis

```bash
# SSH into the control plane node
ssh user@control-plane-node

# Check if apiserver is running
crictl ps | grep kube-apiserver
# If not running, check why:
crictl ps -a | grep kube-apiserver    # look for recently exited containers

# Check apiserver logs
crictl logs <apiserver-container-id>
# Or via static pod logs:
cat /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log | tail -50

# Check kubelet (it manages static pods including apiserver)
systemctl status kubelet
journalctl -u kubelet | tail -50

# Check etcd health (apiserver depends on it)
crictl ps | grep etcd
```

### Resolution

```bash
# Fix 1: Restart kubelet (it will restart the static pod)
sudo systemctl restart kubelet

# Fix 2: Check the static pod manifest for errors
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml
# Look for typos, wrong cert paths, wrong etcd endpoint

# Fix 3: If certs expired, renew them
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

---

## 8.3 Certificate Expired

### Symptom
```
Unable to connect to the server: x509: certificate has expired or is not yet valid:
current time 2026-10-31T05:00:00Z is after 2026-01-15T12:00:00Z
```

### Cause
Kubeadm-managed certificates have a default validity of **1 year**. If not renewed, all cluster communication breaks.

### Diagnosis

```bash
# Check all certificate expiration dates
sudo kubeadm certs check-expiration

# Example output:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
# apiserver                  Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
# apiserver-etcd-client      Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
# apiserver-kubelet-client   Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
# controller-manager.conf   Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
# etcd-healthcheck-client    Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
# etcd-peer                  Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
# etcd-server                Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
# front-proxy-client         Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
# scheduler.conf             Jan 15, 2026 12:00 UTC   EXPIRED        ⚠️
```

### Resolution

```bash
# Step 1: Renew all certificates
sudo kubeadm certs renew all

# Step 2: Restart control plane components
sudo systemctl restart kubelet
# Wait 30 seconds for static pods to restart

# Step 3: Update your kubeconfig
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Step 4: Verify
kubectl get nodes
# Should work now!
```

**Prevent future expiry — set up auto-renewal:**

```bash
# Create a cron job to renew certificates monthly
# File: /etc/cron.monthly/k8s-cert-renew
#!/bin/bash
kubeadm certs renew all
systemctl restart kubelet
echo "$(date): Certificates renewed" >> /var/log/k8s-cert-renewal.log
```

```bash
sudo chmod +x /etc/cron.monthly/k8s-cert-renew
```

---

## 8.4 etcd Backup and Restore

### Backup (run regularly!)

```bash
# Create a snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-20261031.db --write-out=table
```

### Restore (disaster recovery)

```bash
# Step 1: Stop the API server and etcd (move manifests)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# Step 2: Remove the old etcd data
sudo rm -rf /var/lib/etcd/member

# Step 3: Restore from snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-20261031.db \
  --data-dir=/var/lib/etcd \
  --name=control-plane-node \
  --initial-cluster=control-plane-node=https://10.0.0.5:2380 \
  --initial-advertise-peer-urls=https://10.0.0.5:2380

# Step 4: Restore the manifests
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Step 5: Restart kubelet
sudo systemctl restart kubelet

# Step 6: Verify
kubectl get nodes
kubectl get pods -A
```

---

## 8.5 Control Plane Component Crashed

### Quick checks for each component

```bash
# API Server
crictl logs $(crictl ps -q --name kube-apiserver) | tail -20

# Controller Manager
crictl logs $(crictl ps -q --name kube-controller-manager) | tail -20

# Scheduler
crictl logs $(crictl ps -q --name kube-scheduler) | tail -20

# etcd
crictl logs $(crictl ps -q --name etcd) | tail -20
```

**Common fix: ensure manifests are correct:**
```bash
# All static pod manifests live here:
ls /etc/kubernetes/manifests/
# etcd.yaml
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml

# If a manifest is corrupted, restore from kubeadm defaults:
sudo kubeadm init phase control-plane apiserver
sudo kubeadm init phase control-plane controller-manager
sudo kubeadm init phase control-plane scheduler
sudo kubeadm init phase etcd local
```

---

*[← Back to Index](./notes.md)*
