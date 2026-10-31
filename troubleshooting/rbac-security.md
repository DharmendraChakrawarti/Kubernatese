# 6. RBAC & Security Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 6.1 Forbidden – User Cannot Perform Action

### Symptom
```
Error from server (Forbidden): pods is forbidden:
User "system:serviceaccount:default:my-sa" cannot list resource "pods" in API group "" in the namespace "production"
```

### Cause
- The User or ServiceAccount lacks a ClusterRole/Role binding for the requested action.
- Role exists but RoleBinding is in the wrong namespace.
- ClusterRole exists but only a RoleBinding (not ClusterRoleBinding) was created — limits scope to one namespace.

### Diagnosis

```bash
# Check what a user/SA can do
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa

# Check specific permission
kubectl auth can-i list pods -n production --as=system:serviceaccount:default:my-sa
# no

# List all bindings
kubectl get rolebinding,clusterrolebinding -A | grep my-sa
```

### Resolution

**Example: Grant a ServiceAccount read access to pods in a specific namespace:**

**Step 1 – Create a Role:**
```yaml
# File: rbac/pod-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production               # ✅ scope: only "production" namespace
rules:
  - apiGroups: [""]                   # "" = core API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]   # ✅ read-only access
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]                    # ✅ also allow reading pod logs
```

**Step 2 – Bind the Role to the ServiceAccount:**
```yaml
# File: rbac/pod-reader-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-sa-pod-reader
  namespace: production               # ✅ must match the Role's namespace
subjects:
  - kind: ServiceAccount
    name: my-sa                        # ✅ the ServiceAccount name
    namespace: default                 # ✅ namespace where the SA lives
roleRef:
  kind: Role
  name: pod-reader                     # ✅ matches the Role
  apiGroup: rbac.authorization.k8s.io
```

**For cluster-wide access, use ClusterRole + ClusterRoleBinding:**
```yaml
# File: rbac/cluster-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-reader            # ✅ no namespace — cluster-wide
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-sa-cluster-pod-reader
subjects:
  - kind: ServiceAccount
    name: my-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Apply:**
```bash
kubectl apply -f rbac/
```

**Verify:**
```bash
kubectl auth can-i list pods -n production --as=system:serviceaccount:default:my-sa
# yes
```

---

## 6.2 ServiceAccount Token Not Mounted

### Symptom
Application inside the pod cannot authenticate to the Kubernetes API:
```
Error: the server has asked for the client to provide credentials
```

### Cause
- `automountServiceAccountToken: false` is set on the pod or ServiceAccount.
- The ServiceAccount doesn't exist.
- Using Kubernetes 1.24+ where auto-generated long-lived tokens were removed.

### Diagnosis

```bash
# Check if token is mounted
kubectl exec my-app-xyz -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# Should contain: ca.crt, namespace, token

# If empty or doesn't exist → token not mounted
kubectl get pod my-app-xyz -o yaml | grep automountServiceAccountToken
# automountServiceAccountToken: false   ← ⚠️ that's the problem
```

### Resolution

**❌ Broken:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
automountServiceAccountToken: false    # ⚠️ disables token mounting for all pods using this SA
```

**✅ Fixed – enable token mounting:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
automountServiceAccountToken: true     # ✅ mount token into pods
```

**Or override at the pod level:**
```yaml
    spec:
      serviceAccountName: my-sa
      automountServiceAccountToken: true   # ✅ overrides SA-level setting
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
```

---

## 6.3 Pod Security Standards / Admission Violations

### Symptom
Pod creation fails with:
```
Error: pods "my-app" is forbidden: violates PodSecurity "restricted:latest":
  allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true
```

### Cause
Namespace is labeled with a Pod Security Standard (`restricted`, `baseline`, or `privileged`) and the pod spec violates it.

### Diagnosis

```bash
# Check namespace labels
kubectl get namespace production --show-labels
# Labels: pod-security.kubernetes.io/enforce=restricted
```

### Resolution

**Make the pod compliant with the "restricted" standard:**

**❌ Broken – violates restricted standard:**
```yaml
    spec:
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          # ⚠️ No security context at all → fails restricted standard
```

**✅ Fixed – fully compliant with "restricted":**
```yaml
    spec:
      securityContext:
        runAsNonRoot: true                     # ✅ must not run as root
        seccompProfile:
          type: RuntimeDefault                 # ✅ required by restricted
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          securityContext:
            allowPrivilegeEscalation: false    # ✅ required by restricted
            readOnlyRootFilesystem: true       # ✅ best practice
            runAsNonRoot: true                 # ✅ required
            runAsUser: 1000                    # ✅ specific non-root UID
            capabilities:
              drop:
                - ALL                          # ✅ required: drop all capabilities
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}                         # ✅ writable /tmp since root FS is read-only
```

**Or change the namespace enforcement level:**
```bash
# Warn instead of enforce (allows pods but logs warnings)
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite
```

---

## 6.4 Secrets Best Practices

### Common mistakes and fixes

**❌ Bad: Secret values in plain YAML committed to git:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
stringData:
  password: "myS3cretP@ss"     # ⚠️ visible in plain text in your git repo!
```

**✅ Better: Use Sealed Secrets (encrypts secrets for safe git storage):**
```bash
# Install Sealed Secrets controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Create a SealedSecret from a regular Secret
kubectl create secret generic db-creds \
  --from-literal=password='myS3cretP@ss' \
  --dry-run=client -o yaml \
  | kubeseal --format yaml > sealed-db-creds.yaml

# sealed-db-creds.yaml is safe to commit to git
kubectl apply -f sealed-db-creds.yaml
```

**✅ Better: Use External Secrets Operator (syncs from Vault, AWS SM, etc.):**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-creds
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/production/db
        property: password
```

---

*[← Back to Index](./notes.md)*
