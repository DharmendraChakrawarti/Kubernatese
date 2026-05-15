# Module 12: RBAC, Security & Policies

> **Level:** 🔴 Advanced  
> **Objective:** Secure your cluster with RBAC, ServiceAccounts, Pod Security, and OPA.

---

## 12.1 Authentication

How users/services prove their identity to the API Server.

| Method | Use Case |
|--------|----------|
| **x509 Client Certs** | Admin users, kubeadm clusters |
| **Bearer Tokens** | ServiceAccounts |
| **OIDC** | SSO integration (Google, Azure AD, Okta) |
| **Webhook** | External authentication service |

---

## 12.2 RBAC (Role-Based Access Control)

### Core Concepts

```
WHO (Subject)  →  CAN DO WHAT (Role)  →  ON WHAT (Resources)
User/Group/SA  →  verbs on resources  →  via RoleBinding
```

### Role (Namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]                 # Core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
```

### ClusterRole (Cluster-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "persistentvolumes"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: app-sa
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
- kind: Group
  name: ops-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Verbs

| Verb | HTTP Method | Description |
|------|------------|-------------|
| `get` | GET | Read a single resource |
| `list` | GET (collection) | List resources |
| `watch` | GET (streaming) | Watch for changes |
| `create` | POST | Create resource |
| `update` | PUT | Replace resource |
| `patch` | PATCH | Partial update |
| `delete` | DELETE | Delete resource |
| `deletecollection` | DELETE | Delete multiple |

### RBAC Commands

```bash
# Check permissions
kubectl auth can-i create pods
kubectl auth can-i delete deployments --namespace dev
kubectl auth can-i '*' '*'                              # Full admin check
kubectl auth can-i list pods --as jane                  # Impersonate user
kubectl auth can-i list pods --as system:serviceaccount:dev:app-sa
```

---

## 12.3 ServiceAccounts

Identity for Pods to authenticate with the API Server.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: dev
automountServiceAccountToken: false   # Disable auto-mount for security
```

```yaml
# Use in Pod
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: true
```

```bash
# Create token
kubectl create token app-sa --duration=1h
```

---

## 12.4 Pod Security Standards (PSS)

Replaced the deprecated PodSecurityPolicy (PSP).

### Three Levels

| Level | Description |
|-------|-------------|
| **Privileged** | No restrictions (unrestricted) |
| **Baseline** | Prevents known privilege escalations |
| **Restricted** | Heavily restricted (best practice) |

### Enforcement Modes

| Mode | Behavior |
|------|----------|
| `enforce` | Reject violating pods |
| `audit` | Allow but log violations |
| `warn` | Allow but warn the user |

### Apply via Namespace Labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## 12.5 Security Context (Pod/Container Level)

```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    runAsNonRoot: true
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```

---

## 12.6 Network Policies (Security)

See [Module 07](./07-services-networking.md#76-network-policies) for full details.

**Key principle:** Default deny → explicitly allow required traffic.

---

## 12.7 Secrets Encryption at Rest

```yaml
# /etc/kubernetes/enc/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-32-byte-key>
  - identity: {}
```

```bash
# Add to API Server
--encryption-provider-config=/etc/kubernetes/enc/enc.yaml
```

---

## 12.8 OPA Gatekeeper (Policy Engine)

Enforce custom policies using **Rego** language.

```yaml
# Constraint Template
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("Missing labels: %v", [missing])
      }
---
# Constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Namespace"]
  parameters:
    labels: ["team", "environment"]
```

---

## 12.9 Security Best Practices Checklist

- [ ] Enable RBAC (default since K8s 1.8)
- [ ] Use least-privilege ServiceAccounts
- [ ] Don't run containers as root
- [ ] Set `readOnlyRootFilesystem: true`
- [ ] Drop all capabilities, add only needed ones
- [ ] Enable Pod Security Standards (Restricted)
- [ ] Encrypt Secrets at rest
- [ ] Use Network Policies (default deny)
- [ ] Scan images for vulnerabilities (Trivy, Snyk)
- [ ] Enable audit logging
- [ ] Rotate certificates regularly
- [ ] Use external secret managers (Vault, AWS SM)

---

**Previous:** [← Module 11](./11-scheduling.md) | **Next:** [Module 13 →](./13-ingress-networking.md)
