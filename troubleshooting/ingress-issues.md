# 10. Ingress Issues – Detailed Troubleshooting

[← Back to Index](./notes.md)

---

## 10.1 Ingress Returns 404 or 502

### Symptom
Accessing the Ingress URL returns `404 Not Found` or `502 Bad Gateway`.

### Cause
- Backend Service name or port in the Ingress spec doesn't match the actual Service.
- Ingress controller pod is not running.
- Backend pods are not `Ready` (no endpoints).
- Missing `ingressClassName` — no controller picks up the Ingress resource.
- Path matching issue (`Prefix` vs `Exact` vs `ImplementationSpecific`).

### Diagnosis

```bash
# Step 1: Check Ingress resource
kubectl describe ingress my-ingress
# Look for:
#   Default backend: <default>
#   Rules:
#     Host        Path    Backends
#     ----        ----    --------
#     app.example.com
#                 /       my-svc:80 (10.244.1.5:8080, 10.244.2.6:8080)
#                                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                                    If this shows <none> → no endpoints

# Step 2: Check ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50

# Step 3: Check backend service
kubectl get svc my-svc
kubectl get endpoints my-svc
```

### Resolution

**❌ Broken – service name mismatch:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service          # ⚠️ wrong name — actual Service is "my-svc"
                port:
                  number: 80
```

**✅ Fixed:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /    # ✅ optional: rewrite URL path
spec:
  ingressClassName: nginx                            # ✅ MUST specify which controller
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix                         # ✅ matches / and all sub-paths
            backend:
              service:
                name: my-svc                         # ✅ correct Service name
                port:
                  number: 80                         # ✅ matches Service port (not containerPort)
```

**Complete working example (Ingress + Service + Deployment):**

```yaml
# File: deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app                    # ✅ label must match Service selector
    spec:
      containers:
        - name: my-app
          image: myregistry/my-app:1.0
          ports:
            - containerPort: 8080      # ✅ app listens on 8080
---
# File: service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  selector:
    app: my-app                        # ✅ matches pod label
  ports:
    - port: 80                         # ✅ Service port (what Ingress connects to)
      targetPort: 8080                 # ✅ forwards to container port 8080
---
# File: ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-svc           # ✅ matches Service name
                port:
                  number: 80           # ✅ matches Service port
```

---

## 10.2 TLS / HTTPS Not Working on Ingress

### Symptom
- Browser shows "Not Secure" or certificate error.
- `curl -v https://app.example.com` shows SSL handshake failure.

### Cause
- TLS secret referenced in the Ingress does not exist.
- Certificate in the secret doesn't match the hostname.
- cert-manager failed to issue the certificate.
- Secret is in the wrong namespace.

### Resolution

#### Manual TLS (self-managed certificate)

**Step 1 – Create a TLS secret:**
```bash
# If you have cert files
kubectl create secret tls my-tls-secret \
  --cert=./tls.crt \
  --key=./tls.key \
  -n default
```

**Step 2 – Reference in Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx
  tls:                                  # ✅ ADD TLS BLOCK
    - hosts:
        - app.example.com              # ✅ must match the cert's CN/SAN
      secretName: my-tls-secret        # ✅ must exist in the same namespace
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-svc
                port:
                  number: 80
```

#### Automated TLS with cert-manager

**Step 1 – Install cert-manager:**
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

**Step 2 – Create a ClusterIssuer (Let's Encrypt):**
```yaml
# File: cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx                 # ✅ matches your ingress controller
```

**Step 3 – Annotate the Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # ✅ tells cert-manager to issue a cert
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-secret        # ✅ cert-manager creates this automatically
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-svc
                port:
                  number: 80
```

**Debugging cert-manager:**
```bash
# Check Certificate status
kubectl get certificate
kubectl describe certificate <name>

# Check CertificateRequest
kubectl get certificaterequest
kubectl describe certificaterequest <name>

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager --tail=50
```

---

## 10.3 Path-Based Routing

### Scenario
Multiple services behind a single domain:
- `example.com/api/*` → backend API
- `example.com/*` → frontend

### Resolution

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-path-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"        # ✅ enable regex paths
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /api(/|$)(.*)                          # ✅ match /api and /api/*
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /                                      # ✅ catch-all for frontend
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

**⚠️ Important:** Paths are evaluated in order. More specific paths should come first.

---

## 10.4 Ingress Controller Installation

If no Ingress controller is installed, Ingress resources are created but do nothing.

```bash
# Install NGINX Ingress Controller via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.resources.requests.cpu=100m \
  --set controller.resources.requests.memory=128Mi

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

---

*[← Back to Index](./notes.md)*
