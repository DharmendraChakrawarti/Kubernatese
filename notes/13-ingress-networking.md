# Module 13: Ingress & Advanced Networking

> **Level:** 🔴 Advanced  
> **Objective:** Expose applications externally using Ingress, load balancing, and service mesh.

---

## 13.1 What is Ingress?

**Ingress** manages external HTTP/HTTPS access to Services — provides routing, TLS, and virtual hosting.

```
Internet → Ingress Controller (Nginx/Traefik) → Service → Pods
```

### Ingress vs LoadBalancer Service

| Feature | LoadBalancer | Ingress |
|---------|-------------|---------|
| Protocol | TCP/UDP | HTTP/HTTPS |
| Cost | 1 LB per service ($$$) | 1 LB for all services |
| Path routing | ❌ | ✅ |
| TLS termination | ❌ | ✅ |
| Virtual hosts | ❌ | ✅ |
| Rate limiting | ❌ | ✅ |

---

## 13.2 Ingress Controller (Required)

Ingress resources **don't work** without an Ingress Controller. You must install one.

| Controller | Features |
|-----------|---------|
| **NGINX Ingress** | Most popular, feature-rich |
| **Traefik** | Auto-discovery, Let's Encrypt |
| **HAProxy** | High performance |
| **AWS ALB Ingress** | Native AWS integration |
| **Istio Gateway** | Service mesh integration |
| **Contour** | Envoy-based, Kubernetes-native |

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.0/deploy/static/provider/cloud/deploy.yaml

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

---

## 13.3 Ingress Resources

### Simple Ingress (Single Service)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

### Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Host-Based Routing (Virtual Hosting)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: virtual-host-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### TLS/HTTPS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret       # TLS certificate secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

```bash
# Create TLS secret
kubectl create secret tls tls-secret \
  --cert=tls.crt --key=tls.key
```

### Default Backend

```yaml
spec:
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
```

---

## 13.4 Path Types

| Type | Match Behavior | Example |
|------|---------------|---------|
| **Prefix** | Matches URL path prefix | `/api` matches `/api`, `/api/v1`, `/api/users` |
| **Exact** | Exact path match only | `/api` matches only `/api` |
| **ImplementationSpecific** | Depends on IngressClass | Controller-specific |

---

## 13.5 Common NGINX Ingress Annotations

```yaml
metadata:
  annotations:
    # Rewrites
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    
    # Body size
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    
    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    
    # Websockets
    nginx.ingress.kubernetes.io/websocket-services: "ws-service"
    
    # Sticky sessions
    nginx.ingress.kubernetes.io/affinity: "cookie"
```

---

## 13.6 cert-manager (Auto TLS with Let's Encrypt)

```yaml
# ClusterIssuer
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
          class: nginx
---
# Ingress with auto TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls          # Auto-created by cert-manager
```

---

## 13.7 Gateway API (Next Generation)

The successor to Ingress — more expressive and extensible.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "myapp.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
```

---

## 13.8 Service Mesh (Brief Overview)

| Mesh | Features |
|------|---------|
| **Istio** | Traffic management, security, observability (feature-rich, complex) |
| **Linkerd** | Lightweight, simple, Rust-based proxy |
| **Consul Connect** | HashiCorp ecosystem, multi-cloud |

Service mesh adds: mTLS, traffic splitting, circuit breaking, observability — via **sidecar proxies**.

---

**Previous:** [← Module 12](./12-rbac-security.md) | **Next:** [Module 14 →](./14-helm.md)
