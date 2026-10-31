# 9. Helm Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 9.1 Helm Install / Upgrade Fails

### Symptom
```bash
helm install my-release my-chart -f values.yaml
# Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest:
#        error validating "": error validating data: unknown field "replcia" in io.k8s.api.apps.v1.DeploymentSpec
```

### Cause
- Values file has invalid YAML syntax.
- Typo in a field name in the chart template.
- Required values are missing.
- Resource already exists from a previous failed install.
- Namespace doesn't exist.
- API version deprecated/removed in your Kubernetes version.

### Diagnosis

```bash
# Lint the chart for errors
helm lint my-chart/ -f values.yaml
# ==> WARNING: templates/deployment.yaml: unable to parse YAML

# Render templates without installing (see what Helm would apply)
helm template my-release my-chart/ -f values.yaml > rendered.yaml
# Check rendered.yaml for errors

# Dry run against the cluster (catches API validation errors)
helm install my-release my-chart/ -f values.yaml --dry-run --debug

# Check if release already exists
helm list -a
# NAME         NAMESPACE   REVISION   STATUS   CHART
# my-release   default     1          failed   my-chart-1.0.0   ← ⚠️ failed release exists
```

### Resolution

#### Case A: Fix YAML syntax

**❌ Broken values.yaml:**
```yaml
# File: values.yaml
replicaCount: 3
image:
  repository: myregistry/my-app
  tag: 1.0.0
  pullPolicy: IfNotPresent
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:               # ⚠️ wrong indentation or missing colon can break YAML
  cpu: 250m
  memory: 256Mi
```

**✅ Fixed values.yaml:**
```yaml
# File: values.yaml
replicaCount: 3
image:
  repository: myregistry/my-app
  tag: "1.0.0"                    # ✅ quote version strings to avoid YAML parsing as float
  pullPolicy: IfNotPresent
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:                       # ✅ correct indentation
    cpu: 250m                     # ✅ properly nested under requests
    memory: 256Mi                 # ✅
```

#### Case B: Remove a failed release and retry

```bash
# Uninstall the failed release
helm uninstall my-release

# If that fails, force it
helm uninstall my-release --no-hooks

# Clean up any orphaned secrets
kubectl delete secret -l owner=helm,name=my-release

# Retry
helm install my-release my-chart/ -f values.yaml
```

#### Case C: Namespace doesn't exist

```bash
# Create the namespace first
kubectl create namespace production

# Install into the namespace
helm install my-release my-chart/ -f values.yaml -n production --create-namespace
#                                                               ^^^^^^^^^^^^^^^^
#                                                               ✅ auto-create namespace
```

---

## 9.2 Helm Release Stuck in "pending-install" or "pending-upgrade"

### Symptom
```bash
helm list -a
# NAME         STATUS            CHART
# my-release   pending-install   my-chart-1.0.0
```

Helm refuses to install or upgrade because the release is in a broken state.

### Cause
A previous `helm install` or `helm upgrade` was interrupted (timeout, Ctrl+C, network issue).

### Resolution

```bash
# Step 1: Check release history
helm history my-release
# REVISION   STATUS            CHART            DESCRIPTION
# 1          pending-install   my-chart-1.0.0   Initial install incomplete

# Step 2: Try rolling back
helm rollback my-release 0

# Step 3: If rollback fails, delete the release completely
helm uninstall my-release

# Step 4: Clean up Helm secrets manually (if uninstall also fails)
kubectl get secret -l owner=helm,name=my-release
kubectl delete secret -l owner=helm,name=my-release

# Step 5: Re-install
helm install my-release my-chart/ -f values.yaml
```

---

## 9.3 Helm Upgrade Changed Nothing / Wrong Values

### Symptom
You ran `helm upgrade` but the Deployment didn't change — same image, same config.

### Cause
- You forgot to pass `-f values.yaml` — Helm used the chart's default values.
- Values override precedence: CLI `--set` > `-f values.yaml` > chart `values.yaml`.
- The rendered template is identical to the previous revision.

### Diagnosis

```bash
# Compare what's deployed vs. what you expect
helm get values my-release         # currently deployed values
helm get values my-release --all   # including defaults

# Compare two revisions
helm diff revision my-release 2 3  # requires helm-diff plugin

# See the rendered manifests
helm get manifest my-release
```

### Resolution

```bash
# Upgrade with explicit values file
helm upgrade my-release my-chart/ \
  -f values.yaml \
  -f values-production.yaml \         # ✅ environment-specific overrides
  --set image.tag=2.0.0               # ✅ CLI override for the image tag

# Force a re-deploy even if template is identical (e.g., to restart pods)
helm upgrade my-release my-chart/ -f values.yaml --recreate-pods  # deprecated
# Modern approach — add an annotation that changes:
helm upgrade my-release my-chart/ -f values.yaml \
  --set podAnnotations.rollme=$(date +%s)   # ✅ forces a rollout
```

---

## 9.4 Helm Chart Template Debugging

### Useful commands

```bash
# Render templates locally without installing
helm template my-release my-chart/ -f values.yaml

# Render a specific template
helm template my-release my-chart/ -f values.yaml -s templates/deployment.yaml

# Show all computed values
helm template my-release my-chart/ -f values.yaml --show-only templates/deployment.yaml --debug

# Validate against the cluster
helm install my-release my-chart/ -f values.yaml --dry-run --debug 2>&1 | head -100
```

### Example: Debugging a template

**Chart template (`templates/deployment.yaml`):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          {{- if .Values.resources }}         # ← conditional block
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- end }}
```

**values.yaml:**
```yaml
replicaCount: 3

image:
  repository: myregistry/my-app
  tag: "2.0.0"

service:
  port: 8080

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

**Render and verify:**
```bash
helm template my-release ./my-chart -f values.yaml
# Check the output YAML is valid and has the values you expect
```

---

*[← Back to Index](./notes.md)*
