# Module 06: Workload Controllers

> **Level:** 🟡 Intermediate  
> **Objective:** Master all workload controllers — ReplicaSet, Deployment, DaemonSet, StatefulSet, Job, CronJob.

---

## 6.1 Controller Hierarchy

```
Deployment
    └── ReplicaSet
            └── Pod(s)

StatefulSet → Pod(s) (with stable identity)
DaemonSet   → Pod on every node
Job         → Pod(s) that run to completion
CronJob     → Job on a schedule
```

---

## 6.2 ReplicaSet

Ensures a **specified number of pod replicas** are running at all times.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
```

> **Note:** Never create ReplicaSets directly. Use **Deployments** instead, which manage ReplicaSets for you.

---

## 6.3 Deployment ⭐ (Most Important)

Provides **declarative updates**, **rolling updates**, and **rollbacks**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate           # or Recreate
    rollingUpdate:
      maxSurge: 1                 # Extra pods during update
      maxUnavailable: 0           # Zero downtime
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

### Deployment Strategies

| Strategy | Behavior | Downtime |
|----------|----------|----------|
| **RollingUpdate** | Gradually replaces old pods with new | ❌ No |
| **Recreate** | Kills all old pods first, then creates new | ✅ Yes |

### Rolling Update Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `maxSurge` | Max extra pods above desired count | 25% |
| `maxUnavailable` | Max pods that can be unavailable | 25% |

### Rollout Commands

```bash
# Update image
kubectl set image deployment/my-app app=nginx:1.26

# Watch rollout progress
kubectl rollout status deployment/my-app

# View rollout history
kubectl rollout history deployment/my-app
kubectl rollout history deployment/my-app --revision=2

# Rollback
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=1

# Pause/resume rollout
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app

# Scale
kubectl scale deployment/my-app --replicas=5

# Record change cause
kubectl annotate deployment/my-app kubernetes.io/change-cause="Updated to v1.26"
```

### Rollout Visualization

```
v1: [Pod1-v1] [Pod2-v1] [Pod3-v1]           # Initial state

RollingUpdate (maxSurge=1, maxUnavailable=0):
Step 1: [Pod1-v1] [Pod2-v1] [Pod3-v1] [Pod4-v2]   # +1 new
Step 2: [Pod2-v1] [Pod3-v1] [Pod4-v2] [Pod5-v2]   # -1 old, +1 new  
Step 3: [Pod3-v1] [Pod4-v2] [Pod5-v2] [Pod6-v2]   # -1 old, +1 new
Step 4: [Pod4-v2] [Pod5-v2] [Pod6-v2]              # -1 old, done!
```

---

## 6.4 DaemonSet

Ensures a **Pod runs on every node** (or selected nodes).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      tolerations:               # To run on master nodes too
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: monitor
        image: prom/node-exporter:v1.7.0
        ports:
        - containerPort: 9100
```

**Use cases:** Log collectors (Fluentd), monitoring agents (Prometheus Node Exporter), network plugins (Calico), storage daemons.

---

## 6.5 StatefulSet

For **stateful applications** requiring stable identity and persistent storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless  # Required headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:           # PVC per replica
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

### StatefulSet Guarantees

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random (app-7d9f8b) | **Ordered** (app-0, app-1, app-2) |
| Startup order | Parallel | **Sequential** (0→1→2) |
| Shutdown order | Random | **Reverse** (2→1→0) |
| Storage | Shared or none | **Dedicated PVC per pod** |
| Network ID | Random | **Stable DNS** (pod-0.svc.ns) |

### Headless Service (Required)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None              # Makes it headless
  selector:
    app: postgres
  ports:
  - port: 5432
```

DNS: `postgres-0.postgres-headless.default.svc.cluster.local`

---

## 6.6 Job

Runs Pods to **completion** (batch processing).

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 5        # Total successful pods needed
  parallelism: 2        # Run 2 pods at a time
  backoffLimit: 3       # Max retries
  activeDeadlineSeconds: 300  # Timeout
  ttlSecondsAfterFinished: 100  # Auto-cleanup
  template:
    spec:
      restartPolicy: Never   # Required: Never or OnFailure
      containers:
      - name: migrate
        image: my-app:v1
        command: ["python", "migrate.py"]
```

---

## 6.7 CronJob

Runs Jobs on a **schedule** (cron syntax).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"          # 2 AM daily
  concurrencyPolicy: Forbid      # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 200
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:v1
            command: ["/bin/sh", "-c", "backup.sh"]
```

### Cron Schedule Syntax
```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sun=0)
│ │ │ │ │
* * * * *

Examples:
*/5 * * * *     Every 5 minutes
0 */2 * * *     Every 2 hours
0 9 * * 1-5     9 AM weekdays
0 0 1 * *       Midnight, 1st of month
```

---

## 6.8 Comparison Summary

| Controller | Replicas | Ordered | Persistent ID | Use Case |
|-----------|----------|---------|---------------|----------|
| ReplicaSet | ✅ | ❌ | ❌ | (Internal, used by Deployment) |
| **Deployment** | ✅ | ❌ | ❌ | Stateless apps (web servers, APIs) |
| **DaemonSet** | 1/node | ❌ | ❌ | Node agents (monitoring, logging) |
| **StatefulSet** | ✅ | ✅ | ✅ | Databases, message queues |
| **Job** | ✅ | ❌ | ❌ | Batch processing, migrations |
| **CronJob** | ✅ | ❌ | ❌ | Scheduled tasks, backups |

---

**Previous:** [← Module 05](./05-pods.md) | **Next:** [Module 07 →](./07-services-networking.md)
