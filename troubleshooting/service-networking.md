# 2. Service & Networking Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 2.1 Service Has No Endpoints

### Symptom
Traffic to the Service gets no response. `kubectl get endpoints` returns empty:

```bash
kubectl get endpoints my-svc
# NAME     ENDPOINTS   AGE
# my-svc   <none>      5m
```

### Cause
- Service `selector` labels **don't match** any pod's labels.
- All matching pods are not in `Ready` state (failing readiness probe).
- Pods are in a **different namespace** than the Service.

### Diagnosis

```bash
# Step 1: Check the service selector
kubectl get svc my-svc -o jsonpath='{.spec.selector}'
# Output: {"app":"my-app"}

# Step 2: Check pod labels
kubectl get pods --show-labels
# NAME           READY   STATUS    LABELS
# my-app-xyz     1/1     Running   app=myapp    ← "myapp" ≠ "my-app"  ⚠️

# Step 3: Search for pods matching the selector
kubectl get pods -l app=my-app
# No resources found   ← confirms mismatch
```

### Resolution

**❌ Broken – label mismatch:**
```yaml
# File: service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  selector:
    app: my-app          # ⚠️ looking for label "app: my-app"
  ports:
    - port: 80
      targetPort: 8080
---
# File: deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: myapp          # ⚠️ label is "app: myapp" (missing hyphen)
  template:
    metadata:
      labels:
        app: myapp        # ⚠️ DOES NOT match service selector
    spec:
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          ports:
            - containerPort: 8080
```

**✅ Fixed – labels match exactly:**
```yaml
# File: service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  selector:
    app: my-app          # ✅ selector
  ports:
    - port: 80
      targetPort: 8080
---
# File: deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app        # ✅ matches service selector
  template:
    metadata:
      labels:
        app: my-app      # ✅ matches service selector
    spec:
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          ports:
            - containerPort: 8080
```

---

## 2.2 Cannot Access ClusterIP Service from Outside the Cluster

### Symptom
Service works within the cluster (`kubectl exec ... -- curl my-svc:80`) but cannot be reached from your laptop or external network.

### Cause
`ClusterIP` is an internal-only IP address. It's only routable from within the cluster.

### Resolution

#### Option A: Use NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  type: NodePort             # ✅ changed from ClusterIP
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080        # ✅ accessible at <any-node-ip>:30080
```

#### Option B: Use LoadBalancer (cloud environments)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  type: LoadBalancer         # ✅ cloud provider provisions an external LB
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

#### Option C: Use Ingress (recommended for HTTP/HTTPS)
See [ingress-issues.md](./ingress-issues.md) for full examples.

#### Option D: Quick debugging with port-forward
```bash
kubectl port-forward svc/my-svc 8080:80
# Now access http://localhost:8080 on your machine
```

---

## 2.3 Connection Refused / Timeout Between Pods

### Symptom
One pod cannot reach another pod or service:
```bash
kubectl exec -it frontend-xyz -- curl backend-svc:8080
# curl: (7) Failed to connect to backend-svc port 8080: Connection refused
```

### Cause
- **NetworkPolicy** blocking ingress or egress traffic.
- Target pod is not listening on the expected port.
- Target pod is not `Ready` (readiness probe fails → removed from endpoints).
- CNI plugin (Calico/Flannel/Cilium) is not healthy.
- Port number mismatch between Service `targetPort` and container's `containerPort`.

### Diagnosis

```bash
# Step 1: Check if the target service has endpoints
kubectl get endpoints backend-svc
# If empty → pods aren't matching (see § 2.1)

# Step 2: Check NetworkPolicies
kubectl get networkpolicy -A
kubectl describe networkpolicy -n my-namespace

# Step 3: Check CNI plugin pods
kubectl get pods -n kube-system | grep -E "calico|flannel|weave|cilium"

# Step 4: Test from inside a debug pod
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
# Inside the pod:
curl -v backend-svc.my-namespace.svc.cluster.local:8080
nslookup backend-svc
traceroute backend-svc
```

### Resolution

#### Case A: NetworkPolicy blocking traffic

**❌ Broken – default deny with no allow rule:**
```yaml
# This denies ALL ingress to pods in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: backend
spec:
  podSelector: {}       # applies to ALL pods
  policyTypes:
    - Ingress           # ⚠️ blocks all incoming traffic, including from frontend
```

**✅ Fixed – add an allow rule for the frontend:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend                  # ✅ applies to backend pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: frontend        # ✅ allow from frontend namespace
          podSelector:
            matchLabels:
              app: frontend         # ✅ allow from frontend pods
      ports:
        - protocol: TCP
          port: 8080                # ✅ only on port 8080
```

#### Case B: Port mismatch

**❌ Broken:**
```yaml
# Service
spec:
  ports:
    - port: 80
      targetPort: 3000     # ⚠️ Service sends traffic to port 3000

# Deployment
      containers:
        - name: backend
          ports:
            - containerPort: 8080   # ⚠️ App listens on 8080, not 3000
```

**✅ Fixed:**
```yaml
# Service
spec:
  ports:
    - port: 80
      targetPort: 8080     # ✅ matches containerPort

# Deployment
      containers:
        - name: backend
          ports:
            - containerPort: 8080   # ✅ app listens on 8080
```

---

## 2.4 Service External Traffic Policy Issues

### Symptom
LoadBalancer Service works from some nodes but not others, or you see the wrong client IP.

### Cause
- `externalTrafficPolicy: Cluster` (default) routes traffic to any node, masking the real client IP via SNAT.
- `externalTrafficPolicy: Local` only routes to nodes running the pod — if no pod is on a node, that node returns connection refused.

### Resolution

**If you need client IP preservation:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local      # ✅ preserves client IP
  selector:                          #    but traffic only goes to nodes with pods
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

- Ensure you have at least one pod on every node (use a DaemonSet) **or** let the cloud LB health-check remove nodes without pods.

---

*[← Back to Index](./notes.md)*
