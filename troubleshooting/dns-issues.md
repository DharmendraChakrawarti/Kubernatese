# 7. DNS Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 7.1 DNS Resolution Fails Inside Pods

### Symptom
```bash
kubectl exec -it my-app-xyz -- nslookup backend-svc
# ;; connection timed out; no servers could be reached

kubectl exec -it my-app-xyz -- curl backend-svc:8080
# curl: (6) Could not resolve host: backend-svc
```

### Cause
- CoreDNS pods are not running or are in CrashLoopBackOff.
- `resolv.conf` inside the pod is misconfigured.
- NetworkPolicy blocks DNS traffic (UDP/TCP port 53 to `kube-system`).
- CoreDNS ConfigMap has a syntax error.

### Diagnosis

```bash
# Step 1: Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
# NAME                       READY   STATUS    RESTARTS   AGE
# coredns-5644d7b6d9-abcde   0/1     CrashLoopBackOff   5   10m   ← ⚠️

# Step 2: Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Step 3: Check resolv.conf inside the pod
kubectl exec -it my-app-xyz -- cat /etc/resolv.conf
# Expected output:
# nameserver 10.96.0.10          ← CoreDNS ClusterIP
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# Step 4: Check CoreDNS service
kubectl get svc -n kube-system kube-dns
# NAME       TYPE        CLUSTER-IP   PORT(S)
# kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP,9153/TCP

# Step 5: Test DNS directly
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- nslookup backend-svc.default.svc.cluster.local
```

### Resolution

#### Case A: CoreDNS is crashlooping

```bash
# Check CoreDNS ConfigMap for errors
kubectl get configmap coredns -n kube-system -o yaml
```

**✅ Correct CoreDNS ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lazystart
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {         # ✅ forward external queries to node DNS
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

```bash
# Restart CoreDNS after fixing the ConfigMap
kubectl rollout restart deployment/coredns -n kube-system
```

#### Case B: NetworkPolicy blocks DNS

**✅ Allow DNS egress in your NetworkPolicy:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: default
spec:
  podSelector: {}            # applies to all pods in namespace
  policyTypes:
    - Egress
  egress:
    - to:                    # ✅ allow traffic to CoreDNS
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53           # ✅ DNS
        - protocol: TCP
          port: 53           # ✅ DNS (TCP fallback)
    - to:                    # ✅ allow traffic to other pods/services
        - podSelector: {}
```

---

## 7.2 External DNS Not Resolving

### Symptom
Internal services resolve fine, but external hostnames fail:
```bash
kubectl exec -it my-app-xyz -- nslookup google.com
# ;; connection timed out; no servers could be reached

kubectl exec -it my-app-xyz -- nslookup backend-svc
# Name: backend-svc.default.svc.cluster.local
# Address: 10.99.123.45       ← works fine
```

### Cause
- CoreDNS `forward` directive misconfigured.
- Node's `/etc/resolv.conf` is broken or empty.
- Firewall blocks outbound DNS (UDP 53) from the node.

### Resolution

```bash
# Check node DNS
ssh worker-1
cat /etc/resolv.conf
# Should have valid nameservers like 8.8.8.8 or your company's DNS

# Test DNS from the node itself
nslookup google.com
```

**Fix CoreDNS to use explicit upstream servers instead of node resolv.conf:**

```yaml
# Edit: kubectl edit configmap coredns -n kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        forward . 8.8.8.8 8.8.4.4 {    # ✅ use Google DNS directly
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

---

## 7.3 DNS Lookup Slow (ndots Issue)

### Symptom
DNS lookups for external domains (e.g., `api.example.com`) are slow — taking 2-5 seconds instead of milliseconds.

### Cause
Kubernetes sets `ndots:5` in `/etc/resolv.conf` by default. This means any name with fewer than 5 dots is first tried with the search domains appended:

For `api.example.com` (2 dots < 5), Kubernetes tries:
1. `api.example.com.default.svc.cluster.local` → fails
2. `api.example.com.svc.cluster.local` → fails
3. `api.example.com.cluster.local` → fails
4. `api.example.com` → succeeds (finally!)

That's 3 extra DNS queries, each taking ~100ms.

### Resolution

**Option A: Use FQDN (append trailing dot):**
```yaml
      containers:
        - name: my-app
          env:
            - name: API_URL
              value: "https://api.example.com."    # ✅ trailing dot = absolute name, skip search
```

**Option B: Lower ndots in the pod spec:**
```yaml
    spec:
      dnsConfig:
        options:
          - name: ndots
            value: "2"        # ✅ names with 2+ dots are resolved directly first
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
```

**Option C: Use a custom dnsPolicy:**
```yaml
    spec:
      dnsPolicy: "None"                        # ✅ full control over DNS
      dnsConfig:
        nameservers:
          - 10.96.0.10                          # CoreDNS ClusterIP
        searches:
          - default.svc.cluster.local
          - svc.cluster.local
        options:
          - name: ndots
            value: "2"
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
```

---

## 7.4 Headless Service DNS

### Scenario
You need individual pod DNS names (e.g., for StatefulSets like databases).

### Example

```yaml
# Headless Service (ClusterIP: None)
apiVersion: v1
kind: Service
metadata:
  name: db-svc
spec:
  clusterIP: None                # ✅ headless — no single ClusterIP
  selector:
    app: database
  ports:
    - port: 5432
---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-svc            # ✅ must match the headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
```

**DNS names created:**
```
db-0.db-svc.default.svc.cluster.local    → Pod db-0
db-1.db-svc.default.svc.cluster.local    → Pod db-1
db-2.db-svc.default.svc.cluster.local    → Pod db-2
```

**Test:**
```bash
kubectl exec -it my-app -- nslookup db-0.db-svc
# Address: 10.244.1.15   ← individual pod IP
```

---

*[← Back to Index](./notes.md)*
