# Module 15: Monitoring, Logging & Observability

> **Level:** 🔴 Advanced  
> **Objective:** Set up monitoring with Prometheus/Grafana and logging with EFK/Loki.

---

## 15.1 The Three Pillars of Observability

| Pillar | Tool | Purpose |
|--------|------|---------|
| **Metrics** | Prometheus + Grafana | Numeric measurements over time |
| **Logging** | EFK / Loki + Grafana | Textual event records |
| **Tracing** | Jaeger / Tempo | Request flow across services |

---

## 15.2 Metrics Server

Built-in lightweight metrics (CPU/Memory) for HPA.

```bash
# Install
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Usage
kubectl top nodes
kubectl top pods
kubectl top pods --containers
```

---

## 15.3 Prometheus Stack

### Install with Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.adminPassword=admin123
```

### What Gets Installed

| Component | Purpose |
|-----------|---------|
| **Prometheus** | Metrics collection & storage |
| **Grafana** | Visualization dashboards |
| **Alertmanager** | Alert routing & notifications |
| **Node Exporter** | Node-level metrics |
| **kube-state-metrics** | K8s object metrics |

### Key Prometheus Metrics

```promql
# CPU usage per pod
rate(container_cpu_usage_seconds_total[5m])

# Memory usage
container_memory_working_set_bytes

# Pod restarts
kube_pod_container_status_restarts_total

# Node availability
kube_node_status_condition{condition="Ready",status="true"}

# Deployment replicas
kube_deployment_status_replicas_available
```

### ServiceMonitor (Auto-discover targets)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### PrometheusRule (Alerts)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
spec:
  groups:
  - name: my-app
    rules:
    - alert: HighCPUUsage
      expr: rate(container_cpu_usage_seconds_total{container="app"}[5m]) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage on {{ $labels.pod }}"
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
```

---

## 15.4 Logging

### Option A: EFK Stack (Elasticsearch + Fluentd + Kibana)

```
Pods → Fluentd (DaemonSet) → Elasticsearch → Kibana (UI)
```

### Option B: Loki Stack (Lightweight)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  -n monitoring --create-namespace \
  --set grafana.enabled=true \
  --set promtail.enabled=true
```

```
Pods → Promtail (DaemonSet) → Loki → Grafana (UI)
```

### Viewing Logs

```bash
# kubectl logs
kubectl logs my-pod
kubectl logs my-pod -f                  # Stream
kubectl logs my-pod --previous          # Crashed container
kubectl logs -l app=nginx               # By label
kubectl logs my-pod --since=1h          # Last hour
kubectl logs my-pod --tail=50           # Last 50 lines
kubectl logs my-pod -c sidecar          # Specific container
```

---

## 15.5 HorizontalPodAutoscaler (HPA)

Auto-scale based on metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
```

```bash
# Quick HPA creation
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=70

# Check status
kubectl get hpa
```

---

## 15.6 Vertical Pod Autoscaler (VPA)

Auto-adjusts resource requests/limits.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: Auto        # Off | Initial | Recreate | Auto
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
```

---

## 15.7 Cluster Autoscaler

Automatically adjusts the **number of nodes** in the cluster.

```bash
# AWS EKS example
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1
```

### Autoscaler Hierarchy

```
Cluster Autoscaler (adds/removes nodes)
    ↓
HPA (adjusts pod replicas based on metrics)
    ↓
VPA (adjusts resource requests per pod)
```

---

## 15.8 PodDisruptionBudget (PDB)

Ensure minimum availability during voluntary disruptions.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2              # OR maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

---

**Previous:** [← Module 14](./14-helm.md) | **Next:** [Module 16 →](./16-cicd.md)
