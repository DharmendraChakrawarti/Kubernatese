# 12. Debugging Commands – Quick Reference Cheat Sheet

[← Back to Index](./notes.md)

---

## Pod Debugging

```bash
# List all pods across all namespaces
kubectl get pods -A

# List pods with extra info (node, IP)
kubectl get pods -o wide

# Detailed info and events for a pod
kubectl describe pod <pod-name>

# Logs for the current container
kubectl logs <pod-name>

# Logs for a specific container in a multi-container pod
kubectl logs <pod-name> -c <container-name>

# Logs from the previous crashed container
kubectl logs <pod-name> --previous

# Follow logs in real-time
kubectl logs <pod-name> -f

# Logs from the last 30 minutes
kubectl logs <pod-name> --since=30m

# Logs from all pods matching a label
kubectl logs -l app=my-app --all-containers

# Shell into a running container
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- /bin/bash

# Shell into a specific container
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# Run a one-off debug pod
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
kubectl run debug --rm -it --image=busybox:1.36 -- sh

# Ephemeral debug container (K8s 1.23+)
kubectl debug <pod-name> -it --image=busybox:1.36 --target=<container-name>

# Copy files to/from a pod
kubectl cp <pod-name>:/path/to/file ./local-file
kubectl cp ./local-file <pod-name>:/path/to/file

# Get pod YAML
kubectl get pod <pod-name> -o yaml

# Get a specific field with jsonpath
kubectl get pod <pod-name> -o jsonpath='{.status.phase}'
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'

# CPU and memory usage
kubectl top pod <pod-name>
kubectl top pods --all-namespaces --sort-by=cpu
kubectl top pods --all-namespaces --sort-by=memory
```

---

## Node Debugging

```bash
# List nodes with status
kubectl get nodes

# List nodes with extra info
kubectl get nodes -o wide

# Detailed node info
kubectl describe node <node-name>

# Node resource usage
kubectl top nodes

# Check node conditions
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[-1].type,\
READY:.status.conditions[-1].status

# Check taints
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
TAINTS:.spec.taints

# Check allocatable vs allocated resources
kubectl describe node <node-name> | grep -A 10 "Allocated resources"

# Mark node unschedulable
kubectl cordon <node-name>

# Drain node (evict all pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Make node schedulable again
kubectl uncordon <node-name>

# SSH into a node (if you have access)
kubectl debug node/<node-name> -it --image=busybox:1.36
```

---

## Service & Networking Debugging

```bash
# List all services
kubectl get svc -A

# Check endpoints for a service
kubectl get endpoints <svc-name>

# Detailed service info
kubectl describe svc <svc-name>

# Port-forward to a service
kubectl port-forward svc/<svc-name> <local-port>:<svc-port>

# Port-forward to a specific pod
kubectl port-forward pod/<pod-name> <local-port>:<container-port>

# Test DNS from inside a pod
kubectl exec -it <pod-name> -- nslookup <svc-name>
kubectl exec -it <pod-name> -- nslookup <svc-name>.<namespace>.svc.cluster.local

# Test connectivity from inside a pod
kubectl exec -it <pod-name> -- curl -v <svc-name>:<port>
kubectl exec -it <pod-name> -- wget -qO- <svc-name>:<port>

# Check NetworkPolicies
kubectl get networkpolicy -A
kubectl describe networkpolicy <name>
```

---

## Deployment & ReplicaSet Debugging

```bash
# Check deployment status
kubectl get deployment <name>
kubectl describe deployment <name>

# Check rollout status
kubectl rollout status deployment/<name>

# View rollout history
kubectl rollout history deployment/<name>

# Rollback to previous version
kubectl rollout undo deployment/<name>

# Rollback to a specific revision
kubectl rollout undo deployment/<name> --to-revision=<N>

# Restart all pods (rolling restart)
kubectl rollout restart deployment/<name>

# Scale a deployment
kubectl scale deployment/<name> --replicas=5

# Check ReplicaSets
kubectl get rs -l app=<label>

# Pause/resume a rollout
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
```

---

## Events

```bash
# All events sorted by time
kubectl get events --sort-by=.metadata.creationTimestamp

# Events in a specific namespace
kubectl get events -n <namespace>

# Warning events only
kubectl get events --field-selector type=Warning

# Events for a specific resource
kubectl get events --field-selector involvedObject.name=<pod-name>

# Watch events in real-time
kubectl get events -w
```

---

## ConfigMaps & Secrets

```bash
# List ConfigMaps
kubectl get configmap -n <namespace>

# View ConfigMap data
kubectl describe configmap <name>
kubectl get configmap <name> -o yaml

# Create ConfigMap from literals
kubectl create configmap <name> --from-literal=key1=value1 --from-literal=key2=value2

# Create ConfigMap from file
kubectl create configmap <name> --from-file=config.yaml

# List Secrets
kubectl get secret -n <namespace>

# View Secret data (base64 encoded)
kubectl get secret <name> -o yaml

# Decode a secret value
kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 -d

# Create Secret from literals
kubectl create secret generic <name> --from-literal=password='s3cret'
```

---

## Resource Quotas & Limits

```bash
# Check namespace quotas
kubectl describe resourcequota -n <namespace>

# Check LimitRange
kubectl describe limitrange -n <namespace>

# Get resource usage per namespace
kubectl top pods -n <namespace> --sum
```

---

## RBAC Debugging

```bash
# Check what a user/SA can do
kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<sa-name>

# Check a specific permission
kubectl auth can-i create pods -n <namespace> --as=<user>

# List all roles and bindings
kubectl get roles,rolebindings -n <namespace>
kubectl get clusterroles,clusterrolebindings

# Describe a specific role
kubectl describe role <name> -n <namespace>
kubectl describe clusterrole <name>
```

---

## Cluster Info

```bash
# Cluster info
kubectl cluster-info

# Component statuses (deprecated but sometimes useful)
kubectl get componentstatuses

# API server health
kubectl get --raw /healthz

# API resources available
kubectl api-resources

# API versions available
kubectl api-versions

# Current context and cluster
kubectl config current-context
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# View kubeconfig
kubectl config view
```

---

## Useful One-Liners

```bash
# Find all pods NOT in Running status
kubectl get pods -A --field-selector status.phase!=Running

# Find all pods with high restart counts
kubectl get pods -A -o json | jq -r '.items[] | select(.status.containerStatuses[]?.restartCount > 5) | .metadata.namespace + "/" + .metadata.name + " restarts=" + (.status.containerStatuses[0].restartCount | tostring)'

# Get all images used in the cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u

# Find pods consuming the most memory
kubectl top pods -A --sort-by=memory | head -20

# Find pods consuming the most CPU
kubectl top pods -A --sort-by=cpu | head -20

# Get all events in the last hour
kubectl get events -A --sort-by=.lastTimestamp | tail -30

# Delete all completed/failed pods
kubectl delete pods --field-selector=status.phase==Succeeded -A
kubectl delete pods --field-selector=status.phase==Failed -A

# Get all pods on a specific node
kubectl get pods -A --field-selector spec.nodeName=<node-name>

# Quickly check all DaemonSets are fully scheduled
kubectl get ds -A

# Export all resources in a namespace (backup)
kubectl get all -n <namespace> -o yaml > namespace-backup.yaml
```

---

*[← Back to Index](./notes.md)*
