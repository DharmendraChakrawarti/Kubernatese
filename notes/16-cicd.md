# Module 16: CI/CD with Kubernetes

> **Level:** 🔴 Advanced  
> **Objective:** Implement CI/CD pipelines for Kubernetes with GitOps and traditional tools.

---

## 16.1 CI/CD Overview for Kubernetes

```
Code Push → Build → Test → Container Image → Push to Registry → Deploy to K8s
```

### Two Approaches

| Approach | How | Tools |
|----------|-----|-------|
| **Push-based** | CI pipeline pushes to cluster | Jenkins, GitHub Actions, GitLab CI |
| **Pull-based (GitOps)** | Agent in cluster pulls from Git | ArgoCD, Flux |

---

## 16.2 Container Image Best Practices

```dockerfile
# Use multi-stage builds
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine
RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -s /bin/sh -D appuser
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER appuser
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**Image Best Practices:**
- Use **specific tags** (not `latest`)
- Use **multi-stage** builds for smaller images
- Run as **non-root** user
- Scan for **vulnerabilities** (Trivy, Snyk)
- Use **distroless** or **alpine** base images
- Set `imagePullPolicy: IfNotPresent` (or `Always` for mutable tags)

---

## 16.3 GitHub Actions Pipeline

```yaml
# .github/workflows/deploy.yaml
name: Build and Deploy to K8s

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and Push Image
      uses: docker/build-push-action@v5
      with:
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

    - name: Set up kubectl
      uses: azure/setup-kubectl@v3

    - name: Configure kubeconfig
      run: echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig
      
    - name: Deploy to Kubernetes
      run: |
        kubectl --kubeconfig=kubeconfig set image \
          deployment/my-app app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        kubectl --kubeconfig=kubeconfig rollout status deployment/my-app
```

---

## 16.4 Jenkins Pipeline

```groovy
pipeline {
    agent any
    environment {
        REGISTRY = 'your-registry.com'
        IMAGE = "${REGISTRY}/my-app"
        TAG = "${env.BUILD_NUMBER}"
    }
    stages {
        stage('Build') {
            steps {
                sh "docker build -t ${IMAGE}:${TAG} ."
            }
        }
        stage('Test') {
            steps {
                sh "docker run --rm ${IMAGE}:${TAG} npm test"
            }
        }
        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-creds', 
                    usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "docker login ${REGISTRY} -u ${USER} -p ${PASS}"
                    sh "docker push ${IMAGE}:${TAG}"
                }
            }
        }
        stage('Deploy') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh "kubectl set image deployment/my-app app=${IMAGE}:${TAG}"
                    sh "kubectl rollout status deployment/my-app --timeout=300s"
                }
            }
        }
    }
}
```

---

## 16.5 GitOps with ArgoCD

### Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app-k8s.git
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true               # Delete resources removed from Git
      selfHeal: true            # Fix drift from desired state
    syncOptions:
    - CreateNamespace=true
```

### GitOps Workflow

```
Developer → Push code → CI builds image → Push image → Update K8s manifests in Git
                                                              ↓
                              ArgoCD detects change → Syncs to cluster
```

---

## 16.6 Deployment Strategies in K8s

| Strategy | Method | Risk |
|----------|--------|------|
| **Rolling Update** | Gradual replacement | Low |
| **Blue-Green** | Two environments, switch traffic | Low (needs 2x resources) |
| **Canary** | Small % of traffic to new version | Low |
| **A/B Testing** | Route based on headers/cookies | Low |

### Canary with Ingress (NGINX)

```yaml
# Stable (90% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-stable
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-stable
            port:
              number: 80
---
# Canary (10% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-canary
            port:
              number: 80
```

---

## 16.7 Argo Rollouts (Progressive Delivery)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 80
      - pause: {duration: 5m}
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v2
```

---

**Previous:** [← Module 15](./15-monitoring-logging.md) | **Next:** [Module 17 →](./17-production-best-practices.md)
