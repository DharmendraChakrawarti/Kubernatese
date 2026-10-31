# 5. Deployment & Scaling Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 5.1 Deployment Rollout Stuck

### Symptom
`kubectl rollout status` hangs. The new ReplicaSet has 0 ready pods:

```bash
kubectl rollout status deployment/my-app
# Waiting for deployment "my-app" rollout to finish: 1 old replicas are pending termination...

kubectl get rs
# NAME               DESIRED   CURRENT   READY   AGE
# my-app-7d9f8c6b5   3         3         3       2d    ← old (still running)
# my-app-5c4a8e3f2   1         1         0       5m    ← new (not ready)
```

### Cause
- New pods are in `CrashLoopBackOff` or `ImagePullBackOff` (see [pod-issues.md](./pod-issues.md)).
- Insufficient resources for the new pods while old pods are still running.
- PodDisruptionBudget (PDB) prevents old pods from being terminated.
- New pods fail readiness probe — Deployment waits for them to become `Ready`.
- `progressDeadlineSeconds` (default 600s) hasn't been reached yet.

### Diagnosis

```bash
# Step 1: Check deployment events
kubectl describe deployment my-app

# Step 2: Check the new ReplicaSet's pods
kubectl get rs
NEW_RS=$(kubectl get rs -l app=my-app --sort-by=.metadata.creationTimestamp -o name | tail -1)
kubectl describe $NEW_RS

# Step 3: Check the failing pod
kubectl get pods -l app=my-app
kubectl logs <new-pod-name>
kubectl describe pod <new-pod-name>

# Step 4: Check PDB
kubectl get pdb
```

### Resolution

**Quick fix – rollback to the previous version:**
```bash
kubectl rollout undo deployment/my-app

# Or rollback to a specific revision
kubectl rollout history deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=3
```

**Proper fix – configure the Deployment correctly:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  revisionHistoryLimit: 5                # ✅ keep 5 old ReplicaSets for rollback
  progressDeadlineSeconds: 300           # ✅ fail the rollout after 5 minutes
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                        # ✅ create at most 1 extra pod during update
      maxUnavailable: 0                  # ✅ never kill old pods before new ones are Ready
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
          image: myregistry/my-app:2.0.0
          ports:
            - containerPort: 8080
          readinessProbe:                # ✅ REQUIRED for zero-downtime updates
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

---

## 5.2 HPA Not Scaling

### Symptom
```bash
kubectl get hpa
# NAME     REFERENCE           TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
# my-hpa   Deployment/my-app   <unknown>/50%   2         10        2          5m
#                               ^^^^^^^^^ metrics are unknown
```

### Cause
- **Metrics Server is not installed** or not running.
- Pods don't have `resources.requests` defined — HPA needs requests as the baseline (100% = request value).
- For custom metrics: metrics adapter (Prometheus Adapter, Datadog) not installed.
- HPA `scaleTargetRef` doesn't match the Deployment name.

### Diagnosis

```bash
# Check if Metrics Server is running
kubectl get pods -n kube-system | grep metrics-server

# Test metrics API
kubectl top pods
# If this fails → Metrics Server is not working

# Check HPA events
kubectl describe hpa my-hpa
# Look for: "unable to get metrics for resource cpu"
```

### Resolution

**Step 1: Install Metrics Server (if missing):**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For self-signed cert clusters (minikube, kubeadm), add --kubelet-insecure-tls:
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**Step 2: Ensure pods have resource requests:**

**❌ Broken – no resource requests:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          # ⚠️ No resources block — HPA can't calculate percentage
```

**✅ Fixed:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          resources:
            requests:
              cpu: 250m          # ✅ HPA needs this as baseline
              memory: 256Mi      # ✅
            limits:
              cpu: 500m
              memory: 512Mi
```

**Step 3: Create a proper HPA:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app                 # ✅ must match Deployment name exactly
  minReplicas: 2                 # ✅ never go below 2
  maxReplicas: 10                # ✅ never go above 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # ✅ scale up when CPU > 70% of request
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80   # ✅ scale up when memory > 80% of request
  behavior:                        # ✅ OPTIONAL: control scale-up/down speed
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60        # ✅ add at most 2 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120       # ✅ remove at most 1 pod every 2 minutes
```

---

## 5.3 Rolling Update Causing Downtime

### Symptom
During a rolling update, there's a period where requests fail with 502/503 or connection refused.

### Cause
- `maxUnavailable` is too high — old pods are killed before new pods are ready.
- No readiness probe — Kubernetes considers pods `Ready` immediately (even if the app isn't listening yet).
- No `preStop` hook — pod receives SIGTERM and stops accepting connections before it's removed from the Service endpoints.
- Connections in flight are dropped when the old pod terminates.

### Resolution

**Complete zero-downtime Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1             # ✅ add 1 extra pod during rollout
      maxUnavailable: 0       # ✅ NEVER remove old pods until new ones are Ready
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      terminationGracePeriodSeconds: 60    # ✅ allow 60s for graceful shutdown
      containers:
        - name: my-app
          image: myregistry/my-app:2.0.0
          ports:
            - containerPort: 8080
          readinessProbe:                  # ✅ CRITICAL: gates traffic
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          lifecycle:
            preStop:                       # ✅ CRITICAL: drain connections before shutdown
              exec:
                command:
                  - /bin/sh
                  - -c
                  - "sleep 10"             # ✅ wait 10s for endpoints to update
```

**Why `preStop: sleep 10` matters:**

When Kubernetes terminates a pod, two things happen **simultaneously**:
1. SIGTERM is sent to the container.
2. The pod is removed from Service endpoints.

Step 2 takes a few seconds to propagate. Without the `preStop` sleep, the app might stop before all load balancers/iptables rules are updated, causing dropped requests.

---

## 5.4 Deployment Replicas Not Matching

### Symptom
```bash
kubectl get deployment my-app
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# my-app   1/3     3            1           10m
```
Only 1 out of 3 replicas is Ready.

### Diagnosis
```bash
# Check which pods are not ready
kubectl get pods -l app=my-app
# NAME               READY   STATUS    RESTARTS   AGE
# my-app-abc123      1/1     Running   0          10m   ✅
# my-app-def456      0/1     Pending   0          10m   ⚠️
# my-app-ghi789      0/1     Pending   0          10m   ⚠️

# Check why pods are Pending
kubectl describe pod my-app-def456
```

### Resolution
- If `Pending` due to resources → scale cluster or reduce requests (see § 5.1).
- If `CrashLoopBackOff` → fix the app (see [pod-issues.md](./pod-issues.md) § 1.1).
- If anti-affinity prevents scheduling → adjust affinity rules:

```yaml
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:    # ✅ use "preferred" not "required"
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - my-app
                topologyKey: kubernetes.io/hostname
```

---

*[← Back to Index](./notes.md)*
