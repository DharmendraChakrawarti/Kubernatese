# Module 11: Scheduling & Node Management

> **Level:** 🔴 Advanced  
> **Objective:** Control Pod placement with nodeSelector, affinity, taints, tolerations, and topology.

---

## 11.1 nodeSelector (Simple)

Assign Pods to nodes with specific **labels**.

```bash
# Label a node
kubectl label nodes worker-1 disktype=ssd
kubectl label nodes worker-2 disktype=hdd
```

```yaml
spec:
  nodeSelector:
    disktype: ssd               # Pod only runs on SSD nodes
```

---

## 11.2 Node Affinity (Advanced)

More expressive than nodeSelector — supports soft/hard requirements.

```yaml
spec:
  affinity:
    nodeAffinity:
      # HARD requirement (must match)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In           # In, NotIn, Exists, DoesNotExist, Gt, Lt
            values:
            - us-east-1a
            - us-east-1b
      # SOFT preference (try to match)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
```

---

## 11.3 Pod Affinity & Anti-Affinity

Control Pod placement **relative to other Pods**.

```yaml
spec:
  affinity:
    # Co-locate with cache pods
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: kubernetes.io/hostname

    # Spread away from other web pods
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: topology.kubernetes.io/zone
```

| Type | Effect |
|------|--------|
| **Pod Affinity** | Schedule near matching pods (co-location) |
| **Pod Anti-Affinity** | Schedule away from matching pods (spread) |

---

## 11.4 Taints & Tolerations

**Taints** repel Pods from nodes. **Tolerations** allow Pods to run on tainted nodes.

```bash
# Add taint
kubectl taint nodes worker-1 dedicated=gpu:NoSchedule
kubectl taint nodes worker-2 maintenance=true:NoExecute

# Remove taint
kubectl taint nodes worker-1 dedicated=gpu:NoSchedule-
```

### Taint Effects

| Effect | Behavior |
|--------|----------|
| **NoSchedule** | New pods won't be scheduled (existing stay) |
| **PreferNoSchedule** | Try to avoid, but not guaranteed |
| **NoExecute** | Evicts existing pods + prevents new |

### Tolerations in Pod

```yaml
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  - key: "maintenance"
    operator: "Exists"         # Matches any value
    effect: "NoExecute"
    tolerationSeconds: 3600    # Evict after 1 hour
```

### Common Built-in Taints

| Taint | When Applied |
|-------|-------------|
| `node.kubernetes.io/not-ready` | Node is not ready |
| `node.kubernetes.io/unreachable` | Node is unreachable |
| `node.kubernetes.io/memory-pressure` | Node has memory pressure |
| `node.kubernetes.io/disk-pressure` | Node has disk pressure |
| `node.kubernetes.io/pid-pressure` | Node has PID pressure |
| `node-role.kubernetes.io/control-plane` | Control plane nodes |

---

## 11.5 Topology Spread Constraints

Evenly distribute Pods across failure domains.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                           # Max difference between zones
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule     # or ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web
```

---

## 11.6 nodeName (Direct Assignment)

Bypass the scheduler entirely:

```yaml
spec:
  nodeName: worker-1           # Directly assign to node
```

> ⚠️ Not recommended — bypasses all scheduling logic.

---

## 11.7 Node Management

```bash
# Cordon — prevent new pods (keep existing)
kubectl cordon worker-1

# Drain — evict all pods (for maintenance)
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# Uncordon — allow scheduling again
kubectl uncordon worker-1

# View node details
kubectl describe node worker-1
kubectl top node worker-1
```

---

## 11.8 Custom Schedulers

Run multiple schedulers or a custom scheduler:

```yaml
spec:
  schedulerName: my-custom-scheduler
```

---

## 11.9 Scheduling Decision Flowchart

```
New Pod Created (no nodeName)
    ↓
kube-scheduler detects unscheduled Pod
    ↓
FILTER: Which nodes CAN run it?
├── Sufficient resources? (requests)
├── nodeSelector match?
├── Node affinity match?
├── Taint toleration?
├── PV topology?
└── Topology spread OK?
    ↓
SCORE: Which node is BEST?
├── LeastRequested (spread load)
├── NodeAffinity preference
├── Pod affinity preference
├── Image locality
└── Topology spread score
    ↓
BIND: Assign Pod to highest-scoring node
```

---

**Previous:** [← Module 10](./10-namespaces-resources.md) | **Next:** [Module 12 →](./12-rbac-security.md)
