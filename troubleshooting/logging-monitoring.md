# 11. Logging & Monitoring Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 11.1 kubectl logs Shows No Output

### Symptom
```bash
kubectl logs my-app-xyz
# (empty output)
```

### Cause
- Application writes logs to a **file** instead of stdout/stderr.
- Container hasn't fully started yet.
- Container logging driver issue on the node.
- Container restarted and you're looking at the new (empty) container.

### Diagnosis

```bash
# Check if logs exist but from a previous container
kubectl logs my-app-xyz --previous

# Check if the app writes to a file instead of stdout
kubectl exec -it my-app-xyz -- ls -la /var/log/
kubectl exec -it my-app-xyz -- cat /var/log/app.log

# Check container log path on the node
ssh worker-1
ls /var/log/containers/ | grep my-app
cat /var/log/containers/my-app-xyz_default_my-app-<hash>.log
```

### Resolution

#### Case A: Configure your app to log to stdout

**For Node.js:**
```javascript
// ❌ Bad: logging to file
const fs = require('fs');
const logStream = fs.createWriteStream('/var/log/app.log');

// ✅ Good: logging to stdout/stderr
console.log('Info message');    // → stdout
console.error('Error message'); // → stderr
```

**For Python:**
```python
import logging
import sys

# ✅ Log to stdout
logging.basicConfig(
    stream=sys.stdout,
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s'
)
```

**For Java (logback.xml):**
```xml
<!-- ✅ Log to console (stdout) -->
<configuration>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{ISO8601} %-5level [%thread] %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="CONSOLE" />
  </root>
</configuration>
```

#### Case B: Use a sidecar to stream log files

If you **can't** change the app to log to stdout, use a sidecar container:

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
          image: myregistry/my-app:1.0
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
        - name: log-sidecar                    # ✅ SIDECAR CONTAINER
          image: busybox:1.36
          args:
            - /bin/sh
            - -c
            - "tail -F /var/log/app/app.log"   # ✅ streams file to stdout
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
              readOnly: true
      volumes:
        - name: log-volume
          emptyDir: {}
```

Now you can see logs:
```bash
kubectl logs my-app-xyz -c log-sidecar
```

---

## 11.2 Metrics Server Not Available

### Symptom
```bash
kubectl top pods
# error: Metrics API not available
```

### Cause
- Metrics Server is not installed.
- Metrics Server is crashlooping (common: TLS cert verification fails in self-signed clusters).
- Metrics Server can't reach kubelets.

### Diagnosis

```bash
# Check if Metrics Server exists
kubectl get deployment metrics-server -n kube-system
# Error: deployments.apps "metrics-server" not found   ← not installed

# If installed, check if it's running
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=30
# Common error:
# "unable to fully scrape metrics: x509: cannot validate certificate"
```

### Resolution

**Step 1 – Install Metrics Server:**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Step 2 – Fix TLS error (for kubeadm/minikube/kind clusters):**

```bash
# Add --kubelet-insecure-tls to the Metrics Server deployment
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'
```

**Or edit the full deployment:**
```yaml
# kubectl edit deployment metrics-server -n kube-system
spec:
  template:
    spec:
      containers:
        - name: metrics-server
          image: registry.k8s.io/metrics-server/metrics-server:v0.7.0
          args:
            - --cert-dir=/tmp
            - --secure-port=10250
            - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
            - --kubelet-use-node-status-port
            - --metric-resolution=15s
            - --kubelet-insecure-tls          # ✅ ADD THIS LINE
```

**Step 3 – Verify:**
```bash
kubectl top nodes
# NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# worker-1     250m         12%    1024Mi           50%

kubectl top pods -A
```

---

## 11.3 Setting Up Prometheus + Grafana (kube-prometheus-stack)

### Quick install with Helm

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install the full monitoring stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword='admin123' \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.resources.requests.cpu=500m \
  --set prometheus.prometheusSpec.resources.requests.memory=1Gi
```

### Access Grafana

```bash
# Port-forward to access locally
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
# Open http://localhost:3000
# Username: admin
# Password: admin123
```

### Debugging Prometheus scrape issues

```bash
# Check Prometheus targets
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
# Open http://localhost:9090/targets — check for unhealthy targets

# Check if ServiceMonitor is picking up your app
kubectl get servicemonitor -n monitoring
kubectl describe servicemonitor <name> -n monitoring
```

### Example: Monitor your own app

**Step 1 – Expose metrics in your app** (on `/metrics` endpoint).

**Step 2 – Create a ServiceMonitor:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring             # ✅ must be where Prometheus looks
  labels:
    release: monitoring             # ✅ must match Prometheus's serviceMonitorSelector
spec:
  namespaceSelector:
    matchNames:
      - default                     # ✅ namespace where your app runs
  selector:
    matchLabels:
      app: my-app                   # ✅ must match your Service labels
  endpoints:
    - port: http                    # ✅ must match your Service port name
      path: /metrics
      interval: 30s
```

---

## 11.4 Log Aggregation with Loki

### Common issues

| Symptom | Cause | Resolution |
|---|---|---|
| No logs in Grafana/Loki | Promtail DaemonSet not running | `kubectl get ds -n monitoring` — restart if needed |
| Logs delayed by minutes | Promtail batch size too large or flush interval too long | Reduce `client.batchsize` and `client.batchwait` in Promtail config |
| "too many outstanding requests" | Loki is overwhelmed | Increase Loki replicas or add resource limits |

```bash
# Check Promtail pods
kubectl get ds -n monitoring | grep promtail
kubectl logs -n monitoring -l app=promtail --tail=20

# Check Loki pods
kubectl get pods -n monitoring | grep loki
kubectl logs -n monitoring -l app=loki --tail=20
```

---

*[← Back to Index](./notes.md)*
