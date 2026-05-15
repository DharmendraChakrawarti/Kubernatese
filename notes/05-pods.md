# Module 05: Pods — The Smallest Deployable Unit

> **Level:** 🟡 Intermediate  
> **Objective:** Deep understanding of Pods — lifecycle, multi-container patterns, probes, and resource management.

---

## 5.1 What is a Pod?

- **Smallest deployable unit** in Kubernetes (NOT a container)
- Wraps **one or more containers** that share network and storage
- All containers in a Pod share the same **IP address** and **localhost**
- Pods are **ephemeral** — they are created, run, and die

```
┌─────────────── Pod ────────────────┐
│  IP: 10.244.1.5                    │
│                                    │
│  ┌──────────┐  ┌──────────┐       │
│  │Container │  │Container │       │
│  │  (app)   │  │(sidecar) │       │
│  └──────────┘  └──────────┘       │
│                                    │
│  ┌────────────────────────────┐   │
│  │     Shared Volumes         │   │
│  └────────────────────────────┘   │
└────────────────────────────────────┘
```

---

## 5.2 Pod YAML Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
    version: v1
  annotations:
    description: "My application pod"
spec:
  restartPolicy: Always          # Always | OnFailure | Never
  terminationGracePeriodSeconds: 30
  serviceAccountName: default
  
  # Init containers run first, sequentially
  initContainers:
  - name: init-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
  
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
      name: http
      protocol: TCP
    
    # Resource management
    resources:
      requests:                  # Minimum guaranteed
        cpu: "100m"              # 100 millicores = 0.1 CPU
        memory: "128Mi"
      limits:                    # Maximum allowed
        cpu: "500m"
        memory: "256Mi"
    
    # Environment variables
    env:
    - name: APP_ENV
      value: "production"
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db_password
    
    # Volume mounts
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
    - name: data-volume
      mountPath: /data
    
    # Health checks
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
      failureThreshold: 3
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    
    startupProbe:
      httpGet:
        path: /healthz
        port: 80
      failureThreshold: 30
      periodSeconds: 10

  # Volumes
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: data-volume
    emptyDir: {}
```

---

## 5.3 Pod Lifecycle

```
Pending → Running → Succeeded/Failed
   │         │
   │         └── ContainerCreating → Running → Terminated
   │
   └── Unschedulable (no node fits)
       ImagePullBackOff (bad image)
       CrashLoopBackOff (container keeps crashing)
```

| Phase | Meaning |
|-------|---------|
| **Pending** | Accepted but not yet running (scheduling, pulling images) |
| **Running** | At least one container is running |
| **Succeeded** | All containers terminated successfully (exit 0) |
| **Failed** | At least one container terminated with error |
| **Unknown** | Node communication lost |

### Common Pod Conditions

| Condition | Meaning |
|-----------|---------|
| **PodScheduled** | Pod assigned to a node |
| **Initialized** | Init containers completed |
| **ContainersReady** | All containers passed readiness probes |
| **Ready** | Pod is ready to serve traffic |

---

## 5.4 Health Probes

### Three Types of Probes

| Probe | Purpose | Failed Action |
|-------|---------|---------------|
| **Liveness** | Is the container alive? | **Restart** container |
| **Readiness** | Can it serve traffic? | **Remove** from Service endpoints |
| **Startup** | Has it finished starting? | **Kill & restart** (protects slow-starting apps) |

### Probe Mechanisms

```yaml
# HTTP GET
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Awesome

# TCP Socket
livenessProbe:
  tcpSocket:
    port: 3306

# Exec Command
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy

# gRPC
livenessProbe:
  grpc:
    port: 50051
```

### Probe Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `initialDelaySeconds` | 0 | Wait before first probe |
| `periodSeconds` | 10 | How often to probe |
| `timeoutSeconds` | 1 | Probe timeout |
| `successThreshold` | 1 | Min consecutive successes |
| `failureThreshold` | 3 | Min consecutive failures |

---

## 5.5 Multi-Container Pod Patterns

### 1. Sidecar Pattern
```yaml
# Sidecar: Log shipping
spec:
  containers:
  - name: app
    image: my-app:v1
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  - name: log-shipper
    image: fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir: {}
```

### 2. Ambassador Pattern
```yaml
# Ambassador: Proxy to external DB
spec:
  containers:
  - name: app
    image: my-app:v1
    env:
    - name: DB_HOST
      value: "localhost"        # Talk to ambassador
  - name: db-proxy
    image: envoyproxy/envoy:v1.28
```

### 3. Adapter Pattern
```yaml
# Adapter: Transform metrics format
spec:
  containers:
  - name: app
    image: my-app:v1
  - name: metrics-adapter
    image: prom/statsd-exporter
    ports:
    - containerPort: 9102
```

---

## 5.6 Init Containers

Run **before** app containers, **sequentially**, must **succeed**.

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z postgres-svc 5432; do echo waiting; sleep 2; done']
  - name: migrate-db
    image: my-app:v1
    command: ['python', 'manage.py', 'migrate']
  containers:
  - name: app
    image: my-app:v1
```

Use cases: Wait for dependencies, run migrations, download configs, set permissions.

---

## 5.7 Static Pods

- Managed directly by **kubelet** (not the API Server)
- Defined in `/etc/kubernetes/manifests/` on the node
- Control plane components run as static pods
- **Mirror pods** appear in API but can't be controlled from there

```bash
# Check static pod path
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# staticPodPath: /etc/kubernetes/manifests
```

---

## 5.8 Pod Security Context

```yaml
spec:
  securityContext:               # Pod-level
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: my-app:v1
    securityContext:             # Container-level (overrides pod)
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
```

---

## 5.9 Resource Management

### CPU & Memory Units

| Resource | Unit | Examples |
|----------|------|---------|
| CPU | millicores (m) | `100m` = 0.1 CPU, `1000m` = 1 CPU |
| Memory | bytes | `128Mi` = 128 MiB, `1Gi` = 1 GiB |

### QoS Classes

| QoS Class | Condition | Eviction Priority |
|-----------|-----------|-------------------|
| **Guaranteed** | requests == limits (for all containers) | Last to be evicted |
| **Burstable** | requests < limits | Second priority |
| **BestEffort** | No requests or limits | First to be evicted |

---

**Previous:** [← Module 04](./04-kubectl-commands.md) | **Next:** [Module 06 →](./06-workload-controllers.md)
