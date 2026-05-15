# Module 01: Introduction to Kubernetes & Container Orchestration

> **Level:** 🟢 Beginner  
> **Objective:** Understand what Kubernetes is, why it exists, and the problems it solves.

---

## 1.1 What is Kubernetes?

**Kubernetes** (often abbreviated as **K8s**) is an open-source **container orchestration platform** that automates the deployment, scaling, and management of containerized applications.

- **Origin:** Developed by Google, based on their internal system called **Borg**
- **Donated to:** Cloud Native Computing Foundation (CNCF) in 2014
- **Written in:** Go (Golang)
- **Current Status:** The de-facto standard for container orchestration

### Key Facts

| Attribute | Detail |
|-----------|--------|
| Full Name | Kubernetes (Greek for "helmsman" or "pilot") |
| Abbreviation | K8s (K + 8 letters + s) |
| First Release | June 7, 2014 |
| License | Apache License 2.0 |
| Official Website | https://kubernetes.io |
| GitHub | https://github.com/kubernetes/kubernetes |

---

## 1.2 Why Do We Need Kubernetes?

### The Evolution of Application Deployment

```
Traditional Era    →    Virtual Machine Era    →    Container Era
(Bare Metal)            (Hypervisors)               (Docker/K8s)
```

#### 1. Traditional Deployment (Pre-2000s)
- Applications ran directly on physical servers
- **Problems:** No resource boundaries, dependency conflicts, poor utilization, expensive scaling

#### 2. Virtualized Deployment (2000s–2010s)
- VMs provided isolation via hypervisors (VMware, Hyper-V)
- **Problems:** Heavy resource overhead, slow boot times, OS licensing costs

#### 3. Container Deployment (2013–Present)
- Lightweight, share host OS kernel, fast startup
- **Problems at scale:** How to manage 100s or 1000s of containers?

### The Container Orchestration Problem

When you run containers in production, you need answers to:

| Challenge | What K8s Does |
|-----------|---------------|
| How to deploy across multiple nodes? | **Scheduling** — places containers on appropriate nodes |
| What if a container crashes? | **Self-healing** — restarts failed containers automatically |
| How to scale up during traffic spikes? | **Auto-scaling** — HPA, VPA, Cluster Autoscaler |
| How do containers communicate? | **Service Discovery** — built-in DNS and load balancing |
| How to do zero-downtime deployments? | **Rolling Updates** — gradual rollout with rollback capability |
| How to manage configuration? | **ConfigMaps & Secrets** — externalized config management |
| How to manage persistent data? | **Persistent Volumes** — abstracted storage layer |

---

## 1.3 Kubernetes vs Other Orchestrators

| Feature | Kubernetes | Docker Swarm | Apache Mesos | Nomad |
|---------|-----------|-------------|-------------|-------|
| Complexity | High | Low | Very High | Medium |
| Scalability | 5,000+ nodes | ~1,000 nodes | 10,000+ nodes | 10,000+ nodes |
| Auto-scaling | ✅ Built-in | ❌ Manual | ⚠️ Framework-dependent | ✅ Built-in |
| Self-healing | ✅ Yes | ✅ Limited | ✅ Yes | ✅ Yes |
| Service Discovery | ✅ Built-in DNS | ✅ Built-in | ⚠️ Requires setup | ✅ Consul |
| Load Balancing | ✅ Yes | ✅ Yes | ⚠️ External | ✅ Yes |
| Community | 🏆 Largest | Declining | Declining | Growing |
| Learning Curve | Steep | Gentle | Very Steep | Moderate |
| Production Adoption | 🏆 Industry Standard | Legacy use | Niche | Growing |

> **Verdict:** Kubernetes won the orchestration war. It's the industry standard.

---

## 1.4 Kubernetes Key Concepts — Overview

### Core Terminology

| Term | Definition |
|------|-----------|
| **Cluster** | A set of nodes (machines) running Kubernetes |
| **Node** | A single machine (physical or virtual) in the cluster |
| **Pod** | The smallest deployable unit; wraps one or more containers |
| **Service** | A stable network endpoint to access Pods |
| **Deployment** | Manages ReplicaSets and provides declarative updates |
| **Namespace** | Virtual cluster within a cluster for isolation |
| **ConfigMap** | Stores non-sensitive configuration data |
| **Secret** | Stores sensitive data (passwords, tokens) |
| **Volume** | Persistent storage attached to Pods |
| **Ingress** | Manages external HTTP/HTTPS access to Services |

### The Declarative Model

Kubernetes uses a **declarative** approach — you tell it **what you want**, not **how to do it**.

```yaml
# You declare: "I want 3 replicas of my nginx app"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3          # Desired state
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**Kubernetes continuously reconciles** the current state with the desired state:
```
Desired State (YAML) → API Server → Controller → Current State
         ↑                                            |
         └────────── Reconciliation Loop ─────────────┘
```

---

## 1.5 Kubernetes Distributions & Managed Services

### Managed Kubernetes (Cloud)

| Provider | Service | Notes |
|----------|---------|-------|
| AWS | **EKS** (Elastic Kubernetes Service) | Most popular in enterprise |
| Google Cloud | **GKE** (Google Kubernetes Engine) | Best native K8s experience |
| Azure | **AKS** (Azure Kubernetes Service) | Free control plane |
| DigitalOcean | **DOKS** | Budget-friendly |
| Oracle | **OKE** | Free tier available |
| IBM | **IKS** | Enterprise-focused |

### Self-Managed Kubernetes

| Tool | Use Case |
|------|----------|
| **kubeadm** | Official cluster bootstrapping tool |
| **k3s** | Lightweight K8s for edge/IoT |
| **k0s** | Zero-friction Kubernetes |
| **MicroK8s** | Canonical's minimal K8s |
| **RKE2** | Rancher's hardened K8s |
| **OpenShift** | Red Hat's enterprise K8s platform |

### Local Development

| Tool | Use Case |
|------|----------|
| **Minikube** | Single-node cluster for learning |
| **Kind** | Kubernetes in Docker (for CI/CD testing) |
| **Docker Desktop** | Built-in K8s for Mac/Windows |
| **k3d** | k3s in Docker |

---

## 1.6 Kubernetes API & Resources

Everything in Kubernetes is an **API resource**. The API is organized into groups:

### API Groups

| Group | Resources | Example |
|-------|-----------|---------|
| **Core (v1)** | Pod, Service, ConfigMap, Secret, Namespace, Node, PersistentVolume | `apiVersion: v1` |
| **apps/v1** | Deployment, ReplicaSet, StatefulSet, DaemonSet | `apiVersion: apps/v1` |
| **batch/v1** | Job, CronJob | `apiVersion: batch/v1` |
| **networking.k8s.io/v1** | Ingress, NetworkPolicy | `apiVersion: networking.k8s.io/v1` |
| **rbac.authorization.k8s.io/v1** | Role, ClusterRole, RoleBinding | `apiVersion: rbac.authorization.k8s.io/v1` |
| **storage.k8s.io/v1** | StorageClass, CSIDriver | `apiVersion: storage.k8s.io/v1` |
| **autoscaling/v2** | HorizontalPodAutoscaler | `apiVersion: autoscaling/v2` |
| **policy/v1** | PodDisruptionBudget | `apiVersion: policy/v1` |

### Resource YAML Structure (Universal)

Every Kubernetes resource follows this structure:

```yaml
apiVersion: <group/version>   # API group and version
kind: <ResourceType>           # Type of resource
metadata:                      # Identification
  name: <name>                 # Unique name within namespace
  namespace: <namespace>       # Optional, defaults to "default"
  labels:                      # Key-value pairs for organization
    app: my-app
  annotations:                 # Non-identifying metadata
    description: "My application"
spec:                          # Desired state (varies per resource)
  ...
status:                        # Current state (managed by K8s, read-only)
  ...
```

---

## 1.7 The Kubernetes Ecosystem (CNCF Landscape)

Kubernetes is the center of a massive ecosystem:

```
┌─────────────────────────────────────────────────────────┐
│                    CNCF Ecosystem                        │
│                                                          │
│  Container Runtime: containerd, CRI-O                    │
│  Networking (CNI):  Calico, Cilium, Flannel, Weave       │
│  Storage (CSI):     Rook-Ceph, Longhorn, OpenEBS         │
│  Service Mesh:      Istio, Linkerd, Consul Connect       │
│  Monitoring:        Prometheus, Grafana, Datadog          │
│  Logging:           Fluentd, Loki, ELK Stack             │
│  CI/CD:             ArgoCD, Flux, Tekton, Jenkins X       │
│  Security:          OPA/Gatekeeper, Falco, Trivy          │
│  Package Mgmt:      Helm, Kustomize                       │
│  GitOps:            ArgoCD, Flux                          │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 1.8 Key Takeaways

1. **Kubernetes is a container orchestrator** — it manages the lifecycle of containers at scale
2. **Declarative model** — you define desired state, K8s makes it happen
3. **Self-healing** — automatically restarts, replaces, and reschedules containers
4. **Portable** — runs on any cloud, on-prem, or hybrid environment
5. **Extensible** — custom resources (CRDs) let you extend K8s functionality
6. **Industry standard** — adopted by 96%+ of organizations using containers

---

**Next:** [Module 02 — Kubernetes Architecture Deep Dive →](./02-architecture.md)
