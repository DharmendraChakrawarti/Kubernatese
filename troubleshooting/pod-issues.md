# 1. Pod Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 1.1 CrashLoopBackOff

### Symptom
Pod repeatedly starts, crashes, and restarts. `kubectl get pods` shows:

```
NAME           READY   STATUS             RESTARTS      AGE
my-app-xyz     0/1     CrashLoopBackOff   5 (30s ago)   3m
```

### Cause
- Application inside the container exits with a non-zero exit code.
- Missing environment variables, config files, or secrets required by the app.
- Incorrect container `command` or `args` overriding the image entrypoint.
- Dependency (database, API, message queue) is unreachable at startup.
- Liveness probe fails repeatedly, causing kubelet to restart the container.
- Insufficient memory causing the process to be killed before OOMKilled is reported.

### Diagnosis

```bash
# Step 1: Check the logs from the crashed container
kubectl logs my-app-xyz --previous

# Step 2: Look at events and container state
kubectl describe pod my-app-xyz
# Look for:
#   State:          Waiting
#     Reason:       CrashLoopBackOff
#   Last State:     Terminated
#     Reason:       Error
#     Exit Code:    1          ← non-zero means app crashed

# Step 3: Check the exit code
kubectl get pod my-app-xyz -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
# Common exit codes:
#   1   = General application error
#   127 = Command not found (wrong entrypoint)
#   137 = OOMKilled (SIGKILL)
#   143 = Graceful termination (SIGTERM)
```

### Resolution

#### Case A: Application error (Exit Code 1)
Fix the application code or configuration. Example — the app needs a `DATABASE_URL` env var:

**❌ Broken Deployment (missing env var):**
```yaml
# File: deployments/my-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
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
          image: myregistry/my-app:1.2.0
          ports:
            - containerPort: 8080
          # ⚠️ No env vars — app crashes because DATABASE_URL is missing
```

**✅ Fixed Deployment (env var added):**
```yaml
# File: deployments/my-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
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
          image: myregistry/my-app:1.2.0
          ports:
            - containerPort: 8080
          env:                                          # ✅ ADD THIS BLOCK
            - name: DATABASE_URL                        # ✅
              value: "postgres://db-svc:5432/mydb"      # ✅
            - name: SECRET_KEY                          # ✅
              valueFrom:                                # ✅
                secretKeyRef:                           # ✅
                  name: my-app-secrets                  # ✅
                  key: secret-key                       # ✅
```

#### Case B: Wrong command/entrypoint (Exit Code 127)

**❌ Broken:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.2.0
          command: ["start-server"]   # ⚠️ binary "start-server" doesn't exist in the image
```

**✅ Fixed:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.2.0
          command: ["node"]                # ✅ correct binary
          args: ["server.js"]             # ✅ correct arguments
```

#### Case C: Liveness probe too aggressive

**❌ Broken – probe starts before the app is ready:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.2.0
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 0     # ⚠️ probe starts immediately
            periodSeconds: 3           # ⚠️ checks every 3 seconds
            failureThreshold: 1        # ⚠️ fails after just 1 failure → restart
```

**✅ Fixed – give the app time to start:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.2.0
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30    # ✅ wait 30s before first probe
            periodSeconds: 10          # ✅ check every 10 seconds
            failureThreshold: 3        # ✅ allow 3 failures before restart
          startupProbe:                # ✅ ADD a startup probe for slow-starting apps
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30       # ✅ 30 × 10s = 5 minutes to start
            periodSeconds: 10          # ✅
```

#### Case D: Debug interactively
```bash
# Override the entrypoint to get a shell and debug manually
kubectl run debug-pod \
  --image=myregistry/my-app:1.2.0 \
  --restart=Never \
  --command -- sleep 3600

kubectl exec -it debug-pod -- /bin/sh
# Now you're inside the container — test the app manually
```

---

## 1.2 ImagePullBackOff / ErrImagePull

### Symptom
```
NAME           READY   STATUS             RESTARTS   AGE
my-app-xyz     0/1     ImagePullBackOff   0          2m
```

### Cause
- Image name or tag is incorrect / does not exist in the registry.
- Private registry and no `imagePullSecrets` configured.
- Registry is unreachable (network issue or DNS failure on the node).
- Docker Hub rate limit exceeded (anonymous: 100 pulls/6h, authenticated: 200 pulls/6h).

### Diagnosis

```bash
# Check the exact error message
kubectl describe pod my-app-xyz | grep -A 10 "Events:"
# Example output:
#   Warning  Failed   pull image "myregistry/my-app:v2.0": rpc error:
#            unauthorized: authentication required

# Verify the image exists (from your local machine or CI)
docker pull myregistry/my-app:v2.0
# or
crane digest myregistry/my-app:v2.0
```

### Resolution

#### Case A: Typo in image name or tag

**❌ Broken:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:v2.0   # ⚠️ tag "v2.0" doesn't exist, actual tag is "2.0.0"
```

**✅ Fixed:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:2.0.0  # ✅ correct tag
```

#### Case B: Private registry – missing imagePullSecrets

**Step 1 – Create the registry secret:**
```bash
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.example.com \
  --docker-username=myuser \
  --docker-password='s3cur3p@ss' \
  --docker-email=myuser@example.com \
  -n my-namespace
```

**Step 2 – Reference it in the Deployment:**

**❌ Broken (no imagePullSecrets):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: myregistry.example.com/my-app:2.0.0
      # ⚠️ No imagePullSecrets — kubelet can't authenticate to the registry
```

**✅ Fixed:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      imagePullSecrets:                            # ✅ ADD THIS BLOCK
        - name: regcred                            # ✅ matches the secret we created
      containers:
        - name: my-app
          image: myregistry.example.com/my-app:2.0.0
```

#### Case C: Docker Hub rate limit

```bash
# Check if you're being rate-limited
curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:library/nginx:pull" \
  | jq -r '.token' \
  | xargs -I {} curl -s -H "Authorization: Bearer {}" \
    "https://registry-1.docker.io/v2/library/nginx/manifests/latest" -D - -o /dev/null \
  | grep ratelimit
```

Solution: Create a Docker Hub secret with your credentials (see Case B) or use a mirror registry.

---

## 1.3 Pod Stuck in Pending

### Symptom
```
NAME           READY   STATUS    RESTARTS   AGE
my-app-xyz     0/1     Pending   0          10m
```

### Cause
- Insufficient CPU or memory on any node to satisfy the pod's `resources.requests`.
- No node matches the pod's `nodeSelector`, `affinity`, or `tolerations`.
- PersistentVolumeClaim (PVC) referenced by the pod is not bound.
- ResourceQuota in the namespace is exceeded.

### Diagnosis

```bash
# Step 1: Check events for the reason
kubectl describe pod my-app-xyz
# Look for:
#   "0/3 nodes are available: 3 Insufficient cpu."
#   "0/3 nodes are available: 1 node(s) didn't match node selector."
#   "persistentvolumeclaim 'data-pvc' not found"

# Step 2: Check node available resources
kubectl describe nodes | grep -A 6 "Allocated resources"

# Step 3: Check resource quotas
kubectl describe resourcequota -n my-namespace

# Step 4: Check PVC status
kubectl get pvc -n my-namespace
```

### Resolution

#### Case A: Insufficient resources

**❌ Broken – requesting too much:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          resources:
            requests:
              cpu: "4"          # ⚠️ requesting 4 CPUs — no node has this much available
              memory: "8Gi"     # ⚠️ requesting 8Gi — exceeds node capacity
```

**✅ Fixed – lower the requests to fit your nodes:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          resources:
            requests:
              cpu: "500m"       # ✅ 0.5 CPU — fits on most nodes
              memory: "512Mi"   # ✅ 512 MiB — reasonable for most apps
            limits:
              cpu: "1"          # ✅ burst up to 1 CPU
              memory: "1Gi"     # ✅ hard cap at 1 GiB
```

#### Case B: nodeSelector doesn't match any node

**❌ Broken:**
```yaml
    spec:
      nodeSelector:
        disktype: ssd           # ⚠️ no node has this label
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
```

**✅ Fix — add the label to a node:**
```bash
# Label the node
kubectl label nodes worker-node-1 disktype=ssd

# Verify
kubectl get nodes --show-labels | grep disktype
```

**Or remove the nodeSelector if it's not needed:**
```yaml
    spec:
      # nodeSelector removed
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
```

---

## 1.4 OOMKilled (Out of Memory)

### Symptom
Pod terminates with reason `OOMKilled`. Exit code is `137`.

```bash
kubectl describe pod my-app-xyz
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137
```

### Cause
- Container uses more memory than its `resources.limits.memory`.
- Memory leak in the application.
- JVM heap not configured properly (Java apps) — JVM defaults may exceed the container limit.

### Diagnosis

```bash
# Confirm OOM
kubectl get pod my-app-xyz -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# Output: OOMKilled

# Check current resource usage (if pod is still running)
kubectl top pod my-app-xyz
# NAME           CPU(cores)   MEMORY(bytes)
# my-app-xyz     50m          480Mi           ← if limit is 512Mi, it's close to OOM
```

### Resolution

**❌ Broken – memory limit too low:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          resources:
            limits:
              memory: "256Mi"   # ⚠️ app needs ~400Mi at peak
```

**✅ Fixed – increase the memory limit:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          resources:
            requests:
              memory: "512Mi"   # ✅ guaranteed memory
            limits:
              memory: "1Gi"     # ✅ increased limit to handle peak usage
```

**For Java applications – set JVM heap correctly:**
```yaml
      containers:
        - name: my-java-app
          image: myregistry/my-java-app:1.0
          resources:
            requests:
              memory: "512Mi"
            limits:
              memory: "1Gi"
          env:
            - name: JAVA_OPTS                                    # ✅ ADD THIS
              value: "-Xms256m -Xmx768m"                         # ✅ heap max < container limit
            # OR use the container-aware JVM flags (Java 10+):
            - name: JDK_JAVA_OPTIONS                             # ✅ alternative
              value: "-XX:MaxRAMPercentage=75.0"                 # ✅ use 75% of container memory
```

---

## 1.5 Pod Stuck in Terminating

### Symptom
Pod shows `Terminating` status for a very long time and never disappears.

```
NAME           READY   STATUS        RESTARTS   AGE
my-app-xyz     1/1     Terminating   0          30m
```

### Cause
- A finalizer on the pod is preventing deletion (waiting for some controller to act).
- The process inside the container doesn't handle `SIGTERM` and kubelet waits for `terminationGracePeriodSeconds` (default: 30s).
- The node is down — kubelet can't confirm the pod is deleted.
- A `preStop` hook is hanging.

### Diagnosis

```bash
# Check for finalizers
kubectl get pod my-app-xyz -o jsonpath='{.metadata.finalizers}'
# Output: ["some-controller/finalizer"]

# Check if the node is reachable
kubectl get node <node-name>
```

### Resolution

#### Case A: Force delete
```bash
kubectl delete pod my-app-xyz --grace-period=0 --force
```

#### Case B: Remove finalizers
```bash
kubectl patch pod my-app-xyz -p '{"metadata":{"finalizers":null}}'
```

#### Case C: App doesn't handle SIGTERM — add a preStop hook

**✅ Add graceful shutdown handling:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          lifecycle:                        # ✅ ADD THIS BLOCK
            preStop:                        # ✅
              exec:                         # ✅
                command:                    # ✅
                  - /bin/sh                 # ✅
                  - -c                      # ✅
                  - "kill -TERM 1 && sleep 5"  # ✅ send SIGTERM to PID 1 and wait
      terminationGracePeriodSeconds: 60     # ✅ give the app 60s to shut down
```

---

## 1.6 CreateContainerConfigError

### Symptom
```
NAME           READY   STATUS                       RESTARTS   AGE
my-app-xyz     0/1     CreateContainerConfigError    0          1m
```

### Cause
- Referenced ConfigMap does not exist in the namespace.
- Referenced Secret does not exist in the namespace.
- A specific key referenced via `configMapKeyRef` or `secretKeyRef` doesn't exist in the ConfigMap/Secret.

### Diagnosis

```bash
kubectl describe pod my-app-xyz
# Events:
#   Warning  Failed  configmap "app-config" not found
```

### Resolution

**Step 1 – Create the missing ConfigMap:**
```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  -n my-namespace
```

**Or from a file:**
```bash
kubectl create configmap app-config \
  --from-file=config.yaml=./config/production.yaml \
  -n my-namespace
```

**Step 2 – Create the missing Secret:**
```bash
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD='s3cur3p@ss' \
  --from-literal=API_KEY='abc123xyz' \
  -n my-namespace
```

**Example Deployment referencing both:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
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
          image: myregistry/my-app:1.0
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config       # ← must exist in the namespace
                  key: APP_ENV           # ← must exist as a key in the ConfigMap
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets      # ← must exist in the namespace
                  key: DB_PASSWORD       # ← must exist as a key in the Secret
          envFrom:                       # Load all keys from a ConfigMap as env vars
            - configMapRef:
                name: app-config
```

---

## 1.7 RunContainerError

### Symptom
```
NAME           READY   STATUS              RESTARTS   AGE
my-app-xyz     0/1     RunContainerError   0          30s
```

### Cause
- Container runtime cannot start the process (e.g., binary not found in the image).
- Wrong user permissions on mounted volumes.
- `securityContext` prevents execution (e.g., `readOnlyRootFilesystem: true` but the app writes to `/tmp`).

### Resolution

**❌ Broken – app writes to root filesystem but it's read-only:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          securityContext:
            readOnlyRootFilesystem: true    # ⚠️ blocks writes to /tmp, /var, etc.
```

**✅ Fixed – mount a writable volume for /tmp:**
```yaml
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          securityContext:
            readOnlyRootFilesystem: true    # ✅ keep it for security
          volumeMounts:
            - name: tmp-dir                 # ✅ ADD THIS
              mountPath: /tmp               # ✅ app can write to /tmp
      volumes:
        - name: tmp-dir                     # ✅ ADD THIS
          emptyDir: {}                      # ✅ ephemeral writable volume
```

---

## 1.8 Init Container Failures

### Symptom
```
NAME           READY   STATUS       RESTARTS   AGE
my-app-xyz     0/1     Init:Error   3          5m
```

### Cause
- Init container command fails (e.g., database migration fails, dependency check fails).
- Init container image doesn't exist.
- Init container is waiting for a dependency that never becomes available.

### Diagnosis

```bash
# Check init container logs
kubectl logs my-app-xyz -c init-db-check

# Describe the pod to see init container status
kubectl describe pod my-app-xyz
```

### Resolution

**Example – init container that waits for a database to be ready:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      initContainers:                              # ✅ INIT CONTAINERS
        - name: wait-for-db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Waiting for database..."
              until nc -z db-service 5432; do       # ✅ check TCP connectivity
                echo "DB not ready, waiting 2s..."
                sleep 2
              done
              echo "DB is ready!"
        - name: run-migrations
          image: myregistry/my-app:1.0
          command: ["python", "manage.py", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_URL
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          ports:
            - containerPort: 8080
```

**Key points:**
- Init containers run **sequentially** — each must succeed before the next starts.
- If an init container fails, Kubernetes restarts the pod (not just the init container).
- Set `restartPolicy: Always` (default for Deployments) to ensure retries.

---

*[← Back to Index](./notes.md)*
