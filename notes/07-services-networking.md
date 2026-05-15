# Module 07: Services & Networking

> **Level:** 🟡 Intermediate  
> **Objective:** Understand Kubernetes networking model, Service types, DNS, and network policies.

---

## 7.1 Kubernetes Networking Model

### Four Networking Requirements

1. **Pod-to-Pod:** Every Pod can communicate with every other Pod without NAT
2. **Pod-to-Service:** Pods access Services via stable ClusterIP/DNS
3. **External-to-Service:** External traffic reaches Services via NodePort/LoadBalancer/Ingress
4. **Pod-to-External:** Pods can access the internet (via node's network)

### IP Address Ranges

| Range | Purpose | Example |
|-------|---------|---------|
| **Node Network** | Physical/VM network | 192.168.1.0/24 |
| **Pod Network (CIDR)** | Pod IP addresses | 10.244.0.0/16 |
| **Service Network** | ClusterIP addresses | 10.96.0.0/12 |

---

## 7.2 Service Types

### ClusterIP (Default)

Internal-only access within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP              # Default
  selector:
    app: my-app
  ports:
  - name: http
    port: 80                   # Service port
    targetPort: 8080           # Container port
    protocol: TCP
```

```
[Pod A] → my-service:80 → ClusterIP (10.96.0.100) → kube-proxy → [Pod B:8080]
```

### NodePort

Exposes on a **static port on every node**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080            # Range: 30000-32767 (optional, auto-assigned)
```

```
[External] → <NodeIP>:30080 → ClusterIP:80 → Pod:8080
```

### LoadBalancer

Creates a **cloud load balancer** (ELB, ALB, etc.).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

```
[Internet] → Cloud LB (public IP) → NodePort → ClusterIP → Pod
```

### ExternalName

DNS alias to an **external service**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com    # CNAME record
```

### Headless Service (clusterIP: None)

Returns **Pod IPs directly** instead of a single ClusterIP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless-svc
spec:
  clusterIP: None
  selector:
    app: my-app
  ports:
  - port: 80
```

Used with StatefulSets for direct Pod addressing.

---

## 7.3 Service Type Comparison

| Type | Internal | External | Use Case |
|------|----------|----------|----------|
| **ClusterIP** | ✅ | ❌ | Internal microservices |
| **NodePort** | ✅ | ✅ (via node IP) | Dev/testing, on-prem |
| **LoadBalancer** | ✅ | ✅ (via LB IP) | Production cloud apps |
| **ExternalName** | ✅ | ❌ | DNS alias to external service |
| **Headless** | ✅ | ❌ | StatefulSets, direct Pod access |

---

## 7.4 DNS in Kubernetes

### Service DNS

| Resource | DNS Format | Example |
|----------|-----------|---------|
| Service | `<svc>.<ns>.svc.cluster.local` | `my-svc.default.svc.cluster.local` |
| Pod (headless) | `<pod>.<svc>.<ns>.svc.cluster.local` | `web-0.my-svc.default.svc.cluster.local` |
| Pod (by IP) | `<ip-dashed>.<ns>.pod.cluster.local` | `10-244-1-5.default.pod.cluster.local` |

```bash
# Test DNS from within a pod
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 --rm -it -- nslookup my-service
kubectl exec -it my-pod -- nslookup kubernetes.default.svc.cluster.local
```

---

## 7.5 Endpoints & EndpointSlices

Services use **Endpoints** to track Pod IPs:

```bash
kubectl get endpoints my-service
# NAME         ENDPOINTS                         AGE
# my-service   10.244.1.5:8080,10.244.2.3:8080   1h

kubectl get endpointslices -l kubernetes.io/service-name=my-service
```

### Manual Endpoints (No Selector)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service      # Must match Service name
subsets:
- addresses:
  - ip: 203.0.113.50
  ports:
  - port: 80
```

---

## 7.6 Network Policies

Control **traffic flow** between Pods (like firewall rules).

> ⚠️ Requires a CNI that supports NetworkPolicy (Calico, Cilium, Weave). Flannel does NOT.

### Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}              # Applies to ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
```

### Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    - namespaceSelector:
        matchLabels:
          env: production
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
```

### NetworkPolicy Rules

| Rule | Meaning |
|------|---------|
| `podSelector` | Select pods by label |
| `namespaceSelector` | Select namespaces by label |
| `ipBlock` | Select by IP CIDR range |
| AND vs OR | Items in same `from:` entry = AND; separate entries = OR |

---

## 7.7 Multi-Port Services

```yaml
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  - name: metrics
    port: 9090
    targetPort: 9090
```

---

## 7.8 Session Affinity

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800     # 3 hours
```

---

**Previous:** [← Module 06](./06-workload-controllers.md) | **Next:** [Module 08 →](./08-storage-volumes.md)
