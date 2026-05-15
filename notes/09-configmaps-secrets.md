# Module 09: ConfigMaps & Secrets

> **Level:** 🟡 Intermediate  
> **Objective:** Externalize configuration and manage sensitive data.

---

## 9.1 ConfigMaps

Store **non-sensitive** configuration data as key-value pairs.

### Creating ConfigMaps

```bash
# From literals
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# From file
kubectl create configmap nginx-config --from-file=nginx.conf
kubectl create configmap app-config --from-file=config-dir/

# From env file
kubectl create configmap app-config --from-env-file=app.env
```

### ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  # Multi-line config file
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;
      location / {
        proxy_pass http://backend:8080;
      }
    }
```

### Using ConfigMaps in Pods

```yaml
# Method 1: Environment variables (single key)
env:
- name: APP_ENVIRONMENT
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_ENV

# Method 2: All keys as env vars
envFrom:
- configMapRef:
    name: app-config

# Method 3: Mount as volume (files)
volumeMounts:
- name: config
  mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: app-config
    items:                       # Optional: select specific keys
    - key: nginx.conf
      path: nginx.conf
```

---

## 9.2 Secrets

Store **sensitive data** (passwords, tokens, keys) — base64-encoded.

### Creating Secrets

```bash
# From literals
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='S3cur3P@ss!'

# From file
kubectl create secret generic tls-secret \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key

# TLS secret
kubectl create secret tls my-tls-secret \
  --cert=server.crt \
  --key=server.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypass \
  --docker-email=me@example.com
```

### Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque                    # Generic secret
data:                           # base64-encoded
  username: YWRtaW4=           # echo -n "admin" | base64
  password: UzNjdXIzUEBzcyE=
---
# Or use stringData (plain text, auto-encoded)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: S3cur3P@ss!
```

### Secret Types

| Type | Usage |
|------|-------|
| `Opaque` | Generic (default) |
| `kubernetes.io/tls` | TLS certificates |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/basic-auth` | Basic authentication |
| `kubernetes.io/ssh-auth` | SSH credentials |
| `kubernetes.io/service-account-token` | SA tokens (auto-created) |

### Using Secrets in Pods

```yaml
# As environment variables
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password

# As volume (files)
volumeMounts:
- name: secrets
  mountPath: /etc/secrets
  readOnly: true
volumes:
- name: secrets
  secret:
    secretName: db-credentials
    defaultMode: 0400

# Docker registry secret
imagePullSecrets:
- name: regcred
```

---

## 9.3 Important Notes

> ⚠️ **Secrets are NOT encrypted by default** — they are only base64-encoded in etcd.

### Securing Secrets

1. **Enable encryption at rest** in API Server:
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
        secret: <base64-encoded-32-byte-key>
  - identity: {}
```

2. **Use external secret managers:** HashiCorp Vault, AWS Secrets Manager, Azure Key Vault
3. **Use RBAC** to restrict Secret access
4. **Use Sealed Secrets** for GitOps (encrypt before committing)

---

## 9.4 ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| Data type | Non-sensitive | Sensitive |
| Encoding | Plain text | Base64 |
| Size limit | 1 MiB | 1 MiB |
| Encryption at rest | No | Optional |
| Mounted permissions | 0644 | 0644 (set 0400) |
| tmpfs mount | No | Yes (default) |

---

## 9.5 Hot Reloading

- **Volume-mounted** ConfigMaps/Secrets update automatically (~1 min)
- **Environment variables** do NOT auto-update (requires Pod restart)
- Use `kubectl rollout restart deployment/my-app` to force reload

---

**Previous:** [← Module 08](./08-storage-volumes.md) | **Next:** [Module 10 →](./10-namespaces-resources.md)
