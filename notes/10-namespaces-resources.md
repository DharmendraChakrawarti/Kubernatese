# Module 10: Namespaces & Resource Management

> **Level:** 🟡 Intermediate  
> **Objective:** Organize clusters with namespaces; control resources with quotas and limits.

---

## 10.1 Namespaces

Virtual clusters within a physical cluster — provides **isolation** and **organization**.

### Default Namespaces

| Namespace | Purpose |
|-----------|---------|
| `default` | Default for resources with no namespace |
| `kube-system` | K8s system components (CoreDNS, kube-proxy, etc.) |
| `kube-public` | Publicly readable, cluster info |
| `kube-node-lease` | Node heartbeat leases |

### Managing Namespaces

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

kubectl get namespaces
kubectl delete namespace dev    # ⚠️ Deletes ALL resources in it

# Set default namespace
kubectl config set-context --current --namespace=dev

# Work in specific namespace
kubectl get pods -n production
kubectl get all -A              # All namespaces
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: backend
```

### Namespace-scoped vs Cluster-scoped Resources

| Namespaced | Cluster-scoped |
|-----------|---------------|
| Pod, Service, Deployment | Node, PersistentVolume |
| ConfigMap, Secret | ClusterRole, ClusterRoleBinding |
| Role, RoleBinding | Namespace, StorageClass |
| PVC, NetworkPolicy | IngressClass, CSIDriver |

```bash
# Check which resources are namespaced
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

---

## 10.2 Resource Quotas

**Limit total resource consumption** in a namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    # Compute
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    
    # Object counts
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
    configmaps: "20"
    secrets: "20"
    replicationcontrollers: "10"
    
    # Storage
    requests.storage: 100Gi
```

```bash
kubectl get resourcequota -n dev
kubectl describe resourcequota compute-quota -n dev
```

> **When ResourceQuota is set**, every Pod MUST have resource requests/limits.

---

## 10.3 LimitRange

**Default and enforce limits** for individual Pods/Containers.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: dev
spec:
  limits:
  # Container-level defaults
  - type: Container
    default:                    # Default limits
      cpu: 500m
      memory: 256Mi
    defaultRequest:             # Default requests
      cpu: 100m
      memory: 128Mi
    min:                        # Minimum allowed
      cpu: 50m
      memory: 64Mi
    max:                        # Maximum allowed
      cpu: "2"
      memory: 2Gi
  # Pod-level limits
  - type: Pod
    max:
      cpu: "4"
      memory: 4Gi
  # PVC limits
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 50Gi
```

---

## 10.4 Resource Requests vs Limits

```yaml
resources:
  requests:          # Guaranteed minimum (used for scheduling)
    cpu: 100m
    memory: 128Mi
  limits:            # Maximum allowed (enforced at runtime)
    cpu: 500m
    memory: 256Mi
```

| Aspect | Requests | Limits |
|--------|----------|--------|
| Purpose | Scheduling guarantee | Runtime ceiling |
| CPU exceeded | N/A | Throttled (not killed) |
| Memory exceeded | N/A | OOMKilled |
| Scheduler uses | ✅ | ❌ |

### QoS Classes (Revisited)

| Class | Condition | Priority |
|-------|-----------|----------|
| **Guaranteed** | All containers: requests == limits | Highest |
| **Burstable** | At least one container: requests set | Medium |
| **BestEffort** | No requests or limits set | Lowest (first evicted) |

---

## 10.5 Pod Priority & Preemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Critical workloads"
---
# Use in Pod
spec:
  priorityClassName: high-priority
```

When cluster is full, **higher-priority Pods evict lower-priority Pods**.

---

## 10.6 Best Practices

1. **Always set resource requests and limits** for production workloads
2. **Use namespaces** to separate environments (dev, staging, prod)
3. **Apply ResourceQuotas** to prevent runaway resource consumption
4. **Use LimitRange** to set sane defaults
5. **Label namespaces** for NetworkPolicy and RBAC targeting
6. **Don't use `default` namespace** for production workloads

---

**Previous:** [← Module 09](./09-configmaps-secrets.md) | **Next:** [Module 11 →](./11-scheduling.md)
