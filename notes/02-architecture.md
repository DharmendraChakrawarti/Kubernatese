# Module 02: Kubernetes Architecture Deep Dive

> **Level:** 🟢 Beginner  
> **Objective:** Understand every component of the Kubernetes architecture.

---

## 2.1 High-Level Architecture

```
┌───────────────────────────────────────────────────────┐
│                  KUBERNETES CLUSTER                    │
│                                                       │
│  ┌──────────────────────┐  ┌───────────────────────┐  │
│  │    CONTROL PLANE      │  │    WORKER NODE 1      │  │
│  │                       │  │  ┌─────┐ ┌─────┐     │  │
│  │  • API Server         │  │  │Pod 1│ │Pod 2│     │  │
│  │  • etcd               │  │  └─────┘ └─────┘     │  │
│  │  • Scheduler          │  │  kubelet | kube-proxy │  │
│  │  • Controller Manager │  │  Container Runtime    │  │
│  └──────────────────────┘  └───────────────────────┘  │
│                             ┌───────────────────────┐  │
│                             │    WORKER NODE 2      │  │
│                             │  ┌─────┐ ┌─────┐     │  │
│                             │  │Pod 3│ │Pod 4│     │  │
│                             │  └─────┘ └─────┘     │  │
│                             └───────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

---

## 2.2 Control Plane Components

### 2.2.1 kube-apiserver
- **Central hub** — all communication goes through it
- REST API endpoint (port **6443**)
- Only component that talks to **etcd** directly
- Handles authentication, authorization, admission control

```
Request Flow:
kubectl → Authentication → Authorization → Admission Control → etcd
```

### 2.2.2 etcd
- **Distributed key-value store** for ALL cluster data
- Uses **Raft** consensus algorithm
- Ports: **2379** (client), **2380** (peer)
- **Backup is critical** for disaster recovery

```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore /backup/snapshot.db \
  --data-dir=/var/lib/etcd-restored
```

### 2.2.3 kube-scheduler
- Decides **WHERE** to place Pods (doesn't run them)
- Two phases: **Filtering** (can it run?) → **Scoring** (best fit?)
- Considers: resource requests, affinity, taints, topology

### 2.2.4 kube-controller-manager
Runs controller loops that reconcile desired vs current state:

| Controller | Purpose |
|-----------|---------|
| Node Controller | Monitors node health |
| Replication Controller | Ensures correct replica count |
| Deployment Controller | Manages rollouts/rollbacks |
| Endpoints Controller | Populates Service endpoints |
| ServiceAccount Controller | Creates default ServiceAccounts |
| Job Controller | Manages Job completion |

### 2.2.5 cloud-controller-manager
- Only on cloud providers (AWS, GCP, Azure)
- Manages cloud load balancers, routes, node VMs

---

## 2.3 Worker Node Components

### 2.3.1 kubelet
- Agent on every node, port **10250**
- Receives PodSpecs from API Server
- Manages container lifecycle via Container Runtime
- Executes health probes (liveness, readiness, startup)

### 2.3.2 kube-proxy
- Maintains network rules for Service routing
- Modes: **iptables** (default), **IPVS** (high scale), userspace (legacy)
- Routes Service ClusterIP traffic → Pod IPs

### 2.3.3 Container Runtime
| Runtime | Description |
|---------|------------|
| **containerd** | Industry standard, most common |
| **CRI-O** | Lightweight, built for K8s |
| **Docker** | ⚠️ Deprecated as runtime since v1.24 |

```
kubelet → CRI → containerd → runc → Container
```

---

## 2.4 Add-on Components

| Component | Purpose |
|-----------|---------|
| **CoreDNS** | Cluster DNS — resolves service names to ClusterIPs |
| **CNI Plugin** | Container networking (Calico, Cilium, Flannel) |
| **Metrics Server** | Resource usage metrics for HPA |

**DNS Format:** `<service>.<namespace>.svc.cluster.local`

---

## 2.5 Pod Creation Flow (End-to-End)

```
1. kubectl apply -f deployment.yaml
2. API Server authenticates/authorizes, stores in etcd
3. Deployment Controller creates ReplicaSet
4. ReplicaSet Controller creates Pod objects
5. Scheduler assigns Pods to Nodes
6. kubelet on assigned node pulls image, starts container
7. kubelet reports Pod status back to API Server
```

---

## 2.6 Ports Reference

| Component | Port | Purpose |
|-----------|------|---------|
| API Server | 6443 | API requests |
| etcd | 2379/2380 | Client/Peer |
| kubelet | 10250 | API |
| Scheduler | 10259 | Health/metrics |
| Controller Manager | 10257 | Health/metrics |
| NodePort range | 30000-32767 | External access |

---

## 2.7 HA Architecture

- **Odd number of masters** (3 or 5) for etcd quorum
- Load balancer in front of API Servers
- Leader election for scheduler/controller manager
- Stacked etcd or External etcd topology

---

**Previous:** [← Module 01](./01-introduction.md) | **Next:** [Module 03 →](./03-setup-installation.md)
