# Module 14: Helm — Kubernetes Package Manager

> **Level:** 🔴 Advanced  
> **Objective:** Package, template, and deploy applications with Helm charts.

---

## 14.1 What is Helm?

**Helm** is the package manager for Kubernetes — like apt/yum for Linux.

| Concept | Description |
|---------|------------|
| **Chart** | Package of K8s manifests (templates + values) |
| **Release** | Instance of a chart deployed to the cluster |
| **Repository** | Collection of charts (like Docker Hub for images) |
| **Values** | Configuration to customize chart deployment |

---

## 14.2 Installing Helm

```bash
# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS
brew install helm

# Windows
choco install kubernetes-helm

# Verify
helm version
```

---

## 14.3 Essential Commands

### Repository Management

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm repo list
helm repo remove bitnami
helm search repo nginx
helm search hub wordpress        # Search Artifact Hub
```

### Installing Charts

```bash
# Basic install
helm install my-release bitnami/nginx

# With custom values
helm install my-release bitnami/nginx --values values.yaml
helm install my-release bitnami/nginx --set replicaCount=3
helm install my-release bitnami/nginx --set service.type=LoadBalancer

# Install in namespace
helm install my-release bitnami/nginx -n production --create-namespace

# Dry run (preview)
helm install my-release bitnami/nginx --dry-run --debug

# Generate manifests only
helm template my-release bitnami/nginx > manifests.yaml
```

### Managing Releases

```bash
helm list                          # List releases
helm list -A                       # All namespaces
helm status my-release
helm get values my-release         # View current values
helm get manifest my-release       # View deployed manifests
helm get all my-release            # Everything

# Upgrade
helm upgrade my-release bitnami/nginx --set replicaCount=5
helm upgrade my-release bitnami/nginx --values new-values.yaml
helm upgrade --install my-release bitnami/nginx  # Install if not exists

# Rollback
helm rollback my-release 1        # Rollback to revision 1
helm history my-release            # View revision history

# Uninstall
helm uninstall my-release
helm uninstall my-release --keep-history
```

---

## 14.4 Creating Your Own Chart

```bash
helm create my-chart
```

### Chart Structure

```
my-chart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default configuration values
├── charts/                 # Dependencies
├── templates/              # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── _helpers.tpl        # Template helpers
│   ├── NOTES.txt           # Post-install notes
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-chart
description: My application Helm chart
type: application              # application or library
version: 1.0.0                 # Chart version
appVersion: "2.0.0"            # App version
dependencies:
- name: postgresql
  version: "12.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
```

### values.yaml

```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
ingress:
  enabled: true
  hostname: myapp.example.com
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
```

### Template Example (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.port }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        {{- if .Values.env }}
        env:
          {{- range $key, $value := .Values.env }}
          - name: {{ $key }}
            value: {{ $value | quote }}
          {{- end }}
        {{- end }}
```

---

## 14.5 Helm Template Functions

| Function | Example | Output |
|----------|---------|--------|
| `quote` | `{{ .Values.name \| quote }}` | `"value"` |
| `upper` | `{{ .Values.name \| upper }}` | `VALUE` |
| `default` | `{{ .Values.name \| default "nginx" }}` | Fallback value |
| `toYaml` | `{{ toYaml .Values.resources }}` | YAML block |
| `nindent` | `{{ ... \| nindent 4 }}` | Indent with newline |
| `include` | `{{ include "chart.name" . }}` | Include named template |
| `required` | `{{ required "msg" .Values.key }}` | Fail if missing |

### Conditionals & Loops

```yaml
# If/else
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# Range (loop)
{{- range .Values.hosts }}
- host: {{ . }}
{{- end }}

# With (scope change)
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

---

## 14.6 Helm vs Kustomize

| Feature | Helm | Kustomize |
|---------|------|-----------|
| Approach | Templating | Overlays/patches |
| Complexity | Higher | Lower |
| Package mgmt | ✅ Charts & repos | ❌ |
| Conditionals | ✅ | ❌ |
| Built into kubectl | ❌ | ✅ (`kubectl apply -k`) |
| Best for | Complex apps, reuse | Simple customization |

---

## 14.7 Package & Publish

```bash
# Package chart
helm package my-chart/

# Lint chart
helm lint my-chart/

# Push to OCI registry
helm push my-chart-1.0.0.tgz oci://registry.example.com/charts

# Install from OCI
helm install my-release oci://registry.example.com/charts/my-chart --version 1.0.0
```

---

**Previous:** [← Module 13](./13-ingress-networking.md) | **Next:** [Module 15 →](./15-monitoring-logging.md)
