# Module 04: kubectl — The Command-Line Bible

> **Level:** 🟢 Beginner  
> **Objective:** Master kubectl commands for every operation.

---

## 4.1 Command Syntax

```
kubectl [command] [TYPE] [NAME] [flags]
```

| Part | Example |
|------|---------|
| command | get, create, apply, delete, describe, logs |
| TYPE | pod, service, deployment, node |
| NAME | my-pod, my-service |
| flags | -n namespace, -o yaml, --all-namespaces |

---

## 4.2 Essential Commands

### Cluster Info
```bash
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
kubectl get componentstatuses       # Deprecated but useful
kubectl api-resources               # List all resource types
kubectl api-versions                # List all API versions
kubectl explain pod.spec.containers # Built-in docs
```

### Creating Resources
```bash
# Declarative (recommended)
kubectl apply -f manifest.yaml
kubectl apply -f ./manifests/       # Apply entire directory
kubectl apply -f https://url.com/manifest.yaml

# Imperative
kubectl create deployment nginx --image=nginx:1.25
kubectl create namespace dev
kubectl create service clusterip my-svc --tcp=80:8080
kubectl run my-pod --image=nginx:1.25 --port=80
kubectl expose deployment nginx --port=80 --type=NodePort
```

### Getting Resources
```bash
kubectl get pods
kubectl get pods -o wide                    # More details
kubectl get pods -o yaml                    # Full YAML
kubectl get pods -o json                    # JSON format
kubectl get pods -A                         # All namespaces
kubectl get pods -n kube-system             # Specific namespace
kubectl get pods --show-labels              # Show labels
kubectl get pods -l app=nginx               # Filter by label
kubectl get pods --field-selector status.phase=Running
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get all                             # All resource types
kubectl get pod,svc,deploy                  # Multiple types

# Custom columns
kubectl get pods -o custom-columns=\
NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
```

### Describing Resources
```bash
kubectl describe pod my-pod
kubectl describe node worker-1
kubectl describe service my-svc
# Shows events, conditions, and detailed info
```

### Editing & Updating
```bash
kubectl edit deployment my-app              # Opens in editor
kubectl set image deployment/my-app app=nginx:1.26
kubectl scale deployment my-app --replicas=5
kubectl rollout status deployment/my-app
kubectl rollout history deployment/my-app
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=2
kubectl patch pod my-pod -p '{"spec":{"containers":[{"name":"app","image":"nginx:1.26"}]}}'
```

### Deleting Resources
```bash
kubectl delete pod my-pod
kubectl delete pod my-pod --grace-period=0 --force
kubectl delete -f manifest.yaml
kubectl delete pods --all
kubectl delete all --all -n dev
```

### Debugging
```bash
kubectl logs my-pod
kubectl logs my-pod -c my-container         # Specific container
kubectl logs my-pod --previous              # Previous instance
kubectl logs my-pod -f                      # Follow/stream
kubectl logs -l app=nginx --all-containers  # By label
kubectl logs my-pod --tail=100              # Last 100 lines
kubectl logs my-pod --since=1h              # Last hour

kubectl exec -it my-pod -- /bin/bash        # Shell into pod
kubectl exec my-pod -- ls /app              # Run command
kubectl exec -it my-pod -c sidecar -- sh    # Specific container

kubectl port-forward pod/my-pod 8080:80     # Pod port-forward
kubectl port-forward svc/my-svc 8080:80     # Service port-forward

kubectl top nodes                           # Node resource usage
kubectl top pods                            # Pod resource usage
kubectl top pods --containers               # Per-container

kubectl get events --sort-by=.lastTimestamp
kubectl get events -n kube-system
```

### Labels & Annotations
```bash
# Labels
kubectl label pod my-pod env=production
kubectl label pod my-pod env-                # Remove label
kubectl label pods --all status=healthy

# Annotations
kubectl annotate pod my-pod description="My app"
kubectl annotate pod my-pod description-     # Remove
```

---

## 4.3 Output Formatting

| Flag | Output |
|------|--------|
| `-o wide` | Additional columns |
| `-o yaml` | YAML format |
| `-o json` | JSON format |
| `-o name` | Resource names only |
| `-o jsonpath='{...}'` | JSONPath expression |
| `-o custom-columns=...` | Custom table |

```bash
# JSONPath examples
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

---

## 4.4 Imperative Commands for Exams (CKA/CKAD)

```bash
# Generate YAML without creating
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment app --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl create service clusterip my-svc --tcp=80:80 --dry-run=client -o yaml
kubectl expose pod nginx --port=80 --type=NodePort --dry-run=client -o yaml

# Quick pod with resource limits
kubectl run nginx --image=nginx --requests='cpu=100m,memory=128Mi' --limits='cpu=200m,memory=256Mi'

# Create configmap/secret
kubectl create configmap my-config --from-literal=key1=val1 --from-file=config.txt
kubectl create secret generic my-secret --from-literal=password=s3cr3t
```

---

## 4.5 Aliases & Productivity

```bash
# Add to ~/.bashrc or ~/.zshrc
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ka='kubectl apply -f'
alias kdel='kubectl delete'
alias kex='kubectl exec -it'

# Enable autocompletion
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
complete -o default -F __start_kubectl k     # For alias
```

---

**Previous:** [← Module 03](./03-setup-installation.md) | **Next:** [Module 05 →](./05-pods.md)
