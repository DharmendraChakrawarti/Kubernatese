# Kubernetes Troubleshooting Guide

A comprehensive reference for common Kubernetes issues, their root causes, and step-by-step resolutions.

---

## Table of Contents

1. [Pod Issues](#1-pod-issues)
2. [Service & Networking Issues](#2-service--networking-issues)
3. [Storage & Volume Issues](#3-storage--volume-issues)
4. [Node Issues](#4-node-issues)
5. [Deployment & Scaling Issues](#5-deployment--scaling-issues)
6. [RBAC & Security Issues](#6-rbac--security-issues)
7. [DNS Issues](#7-dns-issues)
8. [Cluster-Level Issues](#8-cluster-level-issues)
9. [Helm & Package Manager Issues](#9-helm--package-manager-issues)
10. [Ingress Issues](#10-ingress-issues)
11. [Logging & Monitoring Issues](#11-logging--monitoring-issues)
12. [Useful Debugging Commands](#12-useful-debugging-commands)

---

## 1. Pod Issues

### 1.1 CrashLoopBackOff

**Symptom:** Pod repeatedly starts, crashes, and restarts. Status shows `CrashLoopBackOff`.

**Cause:**
- Application inside the container exits with a non-zero exit code.
- Missing environment variables or config files required by the app.
- Incorrect container command or entrypoint.
- Application dependency (e.g., database) is unreachable.
- Liveness probe fails repeatedly.

**Resolution:**
```bash
# Check pod logs for the crash reason
kubectl logs <pod-name> --previous

# Describe the pod for events and state details
kubectl describe pod <pod-name>

# If needed, override the entrypoint to get a shell and debug
kubectl run debug-pod --image=<image> --restart=Never -- sleep 3600
kubectl exec -it debug-pod -- /bin/sh
```
- Fix the application error shown in logs.
- Ensure all required env vars and ConfigMaps/Secrets are mounted.
- Verify liveness probe settings (`initialDelaySeconds`, `periodSeconds`, `failureThreshold`).

---

### 1.2 ImagePullBackOff / ErrImagePull

**Symptom:** Pod is stuck in `ImagePullBackOff` or `ErrImagePull` status.

**Cause:**
- Image name or tag is incorrect / does not exist in the registry.
- Private registry and no `imagePullSecrets` configured.
- Registry is unreachable (network issue or DNS failure).
- Docker Hub rate limits exceeded.

**Resolution:**
```bash
# Check the exact error
kubectl describe pod <pod-name> | grep -A5 Events

# Verify the image exists
docker pull <image-name>:<tag>

# Create a secret for private registry auth
kubectl create secret docker-registry regcred \
  --docker-server=<registry-url> \
  --docker-username=<user> \
  --docker-password=<password> \
  --docker-email=<email>
```
- Ensure the Deployment spec references the correct `imagePullSecrets`.
- Double-check image name and tag for typos.

---

### 1.3 Pod Stuck in Pending

**Symptom:** Pod stays in `Pending` status and never transitions to `Running`.

**Cause:**
- Insufficient CPU or memory on any node to satisfy the pod's resource requests.
- No node matches the pod's `nodeSelector`, affinity, or toleration requirements.
- PersistentVolumeClaim (PVC) is not bound.
- Cluster autoscaler hasn't provisioned new nodes yet.

**Resolution:**
```bash
# Check why the pod is pending
kubectl describe pod <pod-name>
# Look for messages like "Insufficient cpu", "Insufficient memory", "no nodes available"

# Check node resource availability
kubectl describe nodes | grep -A5 "Allocated resources"

# Check PVC status
kubectl get pvc
```
- Lower the resource requests or add more nodes to the cluster.
- Fix nodeSelector / affinity rules.
- Ensure PVC has a matching PV or a StorageClass that supports dynamic provisioning.

---

### 1.4 OOMKilled (Out of Memory)

**Symptom:** Pod is terminated with reason `OOMKilled`. Exit code `137`.

**Cause:**
- Container uses more memory than its `resources.limits.memory`.
- Memory leak in the application.
- JVM heap not configured properly (Java apps).

**Resolution:**
```bash
# Confirm OOM
kubectl describe pod <pod-name> | grep -i oom

# Check resource usage
kubectl top pod <pod-name>
```
- Increase `resources.limits.memory` in the pod spec.
- Profile the application for memory leaks.
- For Java: set `-Xmx` appropriately relative to the container memory limit.

---

### 1.5 Pod Stuck in Terminating

**Symptom:** Pod shows `Terminating` status and never fully disappears.

**Cause:**
- Finalizers preventing deletion.
- Process inside the container doesn't respond to SIGTERM.
- Node is down and kubelet can't confirm deletion.

**Resolution:**
```bash
# Force delete the pod
kubectl delete pod <pod-name> --grace-period=0 --force

# Check for finalizers
kubectl get pod <pod-name> -o jsonpath='{.metadata.finalizers}'

# Remove finalizers if needed
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":null}}'
```

---

### 1.6 CreateContainerConfigError

**Symptom:** Pod status shows `CreateContainerConfigError`.

**Cause:**
- Referenced ConfigMap or Secret does not exist.
- Key referenced in `configMapKeyRef` or `secretKeyRef` does not exist.

**Resolution:**
```bash
kubectl describe pod <pod-name>
# Look for "configmap not found" or "secret not found"

# Verify the ConfigMap/Secret exists
kubectl get configmap <name>
kubectl get secret <name>

# Check the keys inside
kubectl describe configmap <name>
kubectl get secret <name> -o jsonpath='{.data}'
```
- Create the missing ConfigMap/Secret or fix the reference name/key.

---

### 1.7 RunContainerError

**Symptom:** Pod shows `RunContainerError` status.

**Cause:**
- Container runtime (containerd/Docker) cannot start the process.
- Wrong user permissions on mounted volumes.
- Security context prevents execution (e.g., `readOnlyRootFilesystem: true` but app writes to `/`).

**Resolution:**
```bash
kubectl describe pod <pod-name>
# Check the container runtime error message

# Verify security context
kubectl get pod <pod-name> -o yaml | grep -A10 securityContext
```
- Fix volume permissions or adjust `securityContext` (e.g., `runAsUser`, `fsGroup`).

---

## 2. Service & Networking Issues

### 2.1 Service Has No Endpoints

**Symptom:** `kubectl get endpoints <svc-name>` returns an empty list. Traffic to the Service gets no response.

**Cause:**
- Service selector labels don't match any pod labels.
- All matching pods are not in `Ready` state.
- Pods are in a different namespace than the Service.

**Resolution:**
```bash
# Check service selector
kubectl describe svc <svc-name>

# Check pod labels
kubectl get pods --show-labels

# Verify endpoints
kubectl get endpoints <svc-name>
```
- Ensure the Service `selector` labels match the pod template `labels` exactly.

---

### 2.2 Cannot Access ClusterIP Service from Outside

**Symptom:** ClusterIP Service works within the cluster but not from external machines.

**Cause:**
- ClusterIP is only accessible from within the cluster by design.

**Resolution:**
- Use `NodePort` or `LoadBalancer` type Service for external access.
- Use an Ingress resource for HTTP/HTTPS routing.
- For quick debugging: `kubectl port-forward svc/<svc-name> <local-port>:<svc-port>`.

---

### 2.3 Connection Refused / Connection Timeout Between Pods

**Symptom:** One pod cannot reach another pod or service.

**Cause:**
- NetworkPolicy blocking traffic.
- Target pod not listening on the expected port.
- Target pod is not `Ready`.
- Calico/Flannel/CNI plugin issue.

**Resolution:**
```bash
# Test connectivity from inside a pod
kubectl exec -it <source-pod> -- curl <svc-name>:<port>

# Check network policies
kubectl get networkpolicy -A

# Check if the CNI plugin pods are healthy
kubectl get pods -n kube-system | grep -E "calico|flannel|weave|cilium"
```

---

## 3. Storage & Volume Issues

### 3.1 PVC Stuck in Pending

**Symptom:** PersistentVolumeClaim stays in `Pending` status.

**Cause:**
- No PersistentVolume matches the PVC's `storageClassName`, `accessModes`, or `capacity`.
- StorageClass does not support dynamic provisioning.
- Cloud provider provisioner quota exceeded.

**Resolution:**
```bash
kubectl describe pvc <pvc-name>
kubectl get pv
kubectl get storageclass
```
- Ensure a matching PV exists or a StorageClass with a provisioner is configured.
- Check cloud provider quotas for disk creation.

---

### 3.2 Volume Mount Permission Denied

**Symptom:** Container fails with permission errors when reading/writing to a mounted volume.

**Cause:**
- Volume is owned by `root` but the container runs as a non-root user.
- `fsGroup` not set in the pod's security context.

**Resolution:**
```yaml
# Add fsGroup to the pod spec
spec:
  securityContext:
    fsGroup: 1000
    runAsUser: 1000
```
- Alternatively, use an `initContainer` to `chown` the mount directory.

---

### 3.3 Multi-Attach Error for Volumes

**Symptom:** Pod fails to start with `Multi-Attach error for volume`.

**Cause:**
- The PV uses `ReadWriteOnce` access mode but multiple pods on different nodes try to mount it.

**Resolution:**
- Use `ReadWriteMany` (RWX) access mode if supported by the storage backend (e.g., NFS, EFS).
- Ensure pods sharing the same PVC are scheduled on the same node using pod affinity rules.

---

## 4. Node Issues

### 4.1 Node in NotReady State

**Symptom:** `kubectl get nodes` shows a node as `NotReady`.

**Cause:**
- Kubelet service crashed or stopped.
- Node ran out of disk space, memory, or PIDs.
- Network connectivity lost between the node and the control plane.
- Container runtime (containerd/Docker) crashed.

**Resolution:**
```bash
# Check node conditions
kubectl describe node <node-name> | grep -A20 Conditions

# SSH into the node and check kubelet
systemctl status kubelet
journalctl -u kubelet -f

# Check disk space
df -h

# Check container runtime
systemctl status containerd
```
- Restart kubelet: `sudo systemctl restart kubelet`
- Free disk space or increase disk size.
- Restart the container runtime if needed.

---

### 4.2 Node Disk Pressure / Memory Pressure

**Symptom:** Node has `DiskPressure` or `MemoryPressure` condition set to `True`. Pods get evicted.

**Cause:**
- Disk usage exceeds the eviction threshold (default ~85%).
- System or pod memory consumption is too high.

**Resolution:**
```bash
# Check node conditions
kubectl describe node <node-name> | grep -i pressure

# Clean up unused images and containers on the node
crictl rmi --prune
docker system prune -af  # if using Docker

# Check for pods consuming excessive resources
kubectl top pods --all-namespaces --sort-by=memory
```
- Set proper resource limits on pods.
- Add more nodes or increase node disk/memory.

---

### 4.3 Taint Preventing Pod Scheduling

**Symptom:** Pods are not scheduled onto specific nodes.

**Cause:**
- Node has a taint and the pod spec lacks a matching toleration.

**Resolution:**
```bash
# Check taints on a node
kubectl describe node <node-name> | grep Taints

# Add toleration to the pod spec
```
```yaml
tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
```
- Or remove the taint: `kubectl taint nodes <node-name> <key>:<effect>-`

---

## 5. Deployment & Scaling Issues

### 5.1 Deployment Rollout Stuck

**Symptom:** `kubectl rollout status deployment/<name>` hangs. New ReplicaSet has 0 ready pods.

**Cause:**
- New pods are in `CrashLoopBackOff` or `ImagePullBackOff`.
- Insufficient resources for the new pods.
- PDB (PodDisruptionBudget) prevents old pods from being removed.

**Resolution:**
```bash
kubectl rollout status deployment/<name>
kubectl get rs  # check ReplicaSets
kubectl describe deployment <name>

# Rollback if needed
kubectl rollout undo deployment/<name>

# Check PDB
kubectl get pdb
```

---

### 5.2 HPA Not Scaling

**Symptom:** HorizontalPodAutoscaler shows `<unknown>` for current metrics or doesn't scale.

**Cause:**
- Metrics Server is not installed or not running.
- Pods don't have `resources.requests` defined (HPA needs this as baseline).
- Custom metrics adapter not configured (for custom metrics).

**Resolution:**
```bash
kubectl get hpa
kubectl describe hpa <name>

# Check if Metrics Server is running
kubectl get pods -n kube-system | grep metrics-server

# Install Metrics Server if missing
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
- Ensure every container has `resources.requests.cpu` and/or `resources.requests.memory` defined.

---

### 5.3 Rolling Update Causing Downtime

**Symptom:** During a rolling update, there is a period with no healthy pods serving traffic.

**Cause:**
- `maxUnavailable` set too high.
- Readiness probe not configured or too slow.
- Application takes too long to start.

**Resolution:**
```yaml
strategy:
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0  # ensures old pods stay until new ones are Ready
```
- Configure proper readiness probes with appropriate `initialDelaySeconds`.
- Use `preStop` lifecycle hooks for graceful shutdown.

---

## 6. RBAC & Security Issues

### 6.1 Forbidden – User Cannot Perform Action

**Symptom:** `Error from server (Forbidden): pods is forbidden: User "X" cannot list resource "pods"`.

**Cause:**
- User or ServiceAccount lacks the required ClusterRole/Role binding.

**Resolution:**
```bash
# Check what the user can do
kubectl auth can-i --list --as=<user>

# Check existing bindings
kubectl get clusterrolebinding | grep <user-or-sa>
kubectl get rolebinding -n <namespace> | grep <user-or-sa>

# Create a RoleBinding
kubectl create rolebinding <name> \
  --clusterrole=<role> \
  --serviceaccount=<namespace>:<sa-name> \
  -n <namespace>
```

---

### 6.2 ServiceAccount Token Not Mounted

**Symptom:** Application inside the pod cannot authenticate to the Kubernetes API.

**Cause:**
- `automountServiceAccountToken: false` is set in the pod or ServiceAccount spec.
- The ServiceAccount does not exist.

**Resolution:**
```bash
kubectl get sa -n <namespace>
kubectl describe pod <pod-name> | grep -i serviceaccount
```
- Set `automountServiceAccountToken: true` or manually mount the token.
- Ensure the referenced ServiceAccount exists in the namespace.

---

## 7. DNS Issues

### 7.1 DNS Resolution Fails Inside Pods

**Symptom:** `nslookup <service-name>` or `curl <service-name>` fails from within a pod.

**Cause:**
- CoreDNS pods are not running or are in CrashLoopBackOff.
- `resolv.conf` inside the pod is misconfigured.
- NetworkPolicy blocks DNS traffic (UDP port 53).

**Resolution:**
```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS from inside a pod
kubectl exec -it <pod> -- nslookup kubernetes.default

# Check resolv.conf
kubectl exec -it <pod> -- cat /etc/resolv.conf
```
- Restart CoreDNS: `kubectl rollout restart deployment/coredns -n kube-system`
- Ensure NetworkPolicies allow egress to `kube-system` namespace on UDP 53.

---

### 7.2 External DNS Not Resolving

**Symptom:** Pods can resolve internal service names but not external hostnames (e.g., `google.com`).

**Cause:**
- CoreDNS upstream forwarder misconfigured.
- Node DNS (`/etc/resolv.conf` on the node) is broken.
- Firewall blocks outbound DNS (UDP 53).

**Resolution:**
```bash
# Check CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml
# Look for the "forward" directive
```
- Ensure the `forward . /etc/resolv.conf` or `forward . 8.8.8.8 8.8.4.4` directive is present.

---

## 8. Cluster-Level Issues

### 8.1 etcd Unhealthy / Slow

**Symptom:** API server is slow. `kubectl` commands take a long time. etcd health check fails.

**Cause:**
- Disk I/O too slow for etcd (etcd requires fast SSD storage).
- etcd cluster lost quorum (majority of members are down).
- Large number of objects causing high etcd database size.

**Resolution:**
```bash
# Check etcd health (if you have access)
etcdctl endpoint health --cluster

# Check etcd database size
etcdctl endpoint status --write-out=table

# Compact and defrag etcd
etcdctl compact <revision>
etcdctl defrag
```
- Use SSD-backed disks for etcd.
- Ensure odd number of etcd members (1, 3, 5) for quorum.

---

### 8.2 API Server Unavailable

**Symptom:** `kubectl` commands return `connection refused` or timeout. The cluster seems down.

**Cause:**
- `kube-apiserver` pod/process crashed.
- etcd is down (API server depends on it).
- Certificate expired.
- Load balancer in front of API server is misconfigured.

**Resolution:**
```bash
# Check API server pod (in static pod manifest path)
ls /etc/kubernetes/manifests/kube-apiserver.yaml

# Check kube-apiserver logs
crictl logs <apiserver-container-id>
# or
journalctl -u kubelet | grep apiserver

# Check certificate expiry
kubeadm certs check-expiration

# Renew certificates
kubeadm certs renew all
```

---

### 8.3 Certificate Expired

**Symptom:** `x509: certificate has expired or is not yet valid` error in kubectl or component logs.

**Cause:**
- Kubernetes component certificates (default 1-year validity for kubeadm) have expired.

**Resolution:**
```bash
# Check all certificate expirations
kubeadm certs check-expiration

# Renew all certificates
kubeadm certs renew all

# Restart control plane components
systemctl restart kubelet
```

---

## 9. Helm & Package Manager Issues

### 9.1 Helm Install / Upgrade Fails

**Symptom:** `helm install` or `helm upgrade` returns an error.

**Cause:**
- Values file has invalid YAML syntax.
- Required values are missing.
- Resource already exists (from a previous failed install).
- Namespace doesn't exist.

**Resolution:**
```bash
# Lint the chart
helm lint <chart-path> -f values.yaml

# Dry run to preview rendered manifests
helm install <name> <chart> --dry-run --debug -f values.yaml

# If previous install left orphaned resources
helm uninstall <name>
# or fix the release
helm rollback <name> <revision>
```

---

### 9.2 Helm Release Stuck in "pending-install" or "pending-upgrade"

**Symptom:** Helm shows the release in a broken state and won't allow install or upgrade.

**Cause:**
- Previous Helm operation was interrupted (e.g., timeout, user killed the process).

**Resolution:**
```bash
# Check release history
helm history <release-name>

# Rollback to the last successful revision
helm rollback <release-name> <revision>

# If rollback fails, delete the release and re-install
helm uninstall <release-name>
# Then remove any remaining secrets
kubectl delete secret -l owner=helm,name=<release-name>
```

---

## 10. Ingress Issues

### 10.1 Ingress Returns 404 or 502

**Symptom:** Accessing the Ingress URL returns a `404 Not Found` or `502 Bad Gateway`.

**Cause:**
- Backend Service or port doesn't match what's in the Ingress spec.
- Ingress controller pod is not running.
- Backend pods are not Ready.
- Missing `ingressClassName` or annotation for the controller.

**Resolution:**
```bash
# Check Ingress resource
kubectl describe ingress <name>

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Verify backend service exists and has endpoints
kubectl get svc <backend-svc>
kubectl get endpoints <backend-svc>
```

---

### 10.2 TLS / HTTPS Not Working on Ingress

**Symptom:** HTTPS returns certificate errors or the browser shows "Not Secure".

**Cause:**
- TLS secret referenced in the Ingress does not exist.
- Certificate in the secret doesn't match the host.
- cert-manager failed to issue the certificate.

**Resolution:**
```bash
# Check the TLS secret
kubectl get secret <tls-secret-name>
kubectl describe secret <tls-secret-name>

# If using cert-manager, check Certificate resource
kubectl get certificate
kubectl describe certificate <name>

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager
```

---

## 11. Logging & Monitoring Issues

### 11.1 kubectl logs Shows No Output

**Symptom:** `kubectl logs <pod>` returns empty output even though the app should be logging.

**Cause:**
- Application writes logs to a file instead of stdout/stderr.
- Container hasn't started yet.
- Logging driver issue on the node.

**Resolution:**
- Configure the application to log to stdout/stderr.
- Use a sidecar container to tail log files and stream to stdout.
- Check the logging driver on the node: `docker info | grep Logging` or `crictl info`.

---

### 11.2 Metrics Server Not Available

**Symptom:** `kubectl top pods` returns `error: Metrics API not available`.

**Cause:**
- Metrics Server is not installed.
- Metrics Server pod is crashlooping (often TLS issues in self-signed cert clusters).

**Resolution:**
```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# If TLS errors, add --kubelet-insecure-tls flag
kubectl edit deployment metrics-server -n kube-system
# Add to args: --kubelet-insecure-tls
```

---

## 12. Useful Debugging Commands

### Quick Reference

```bash
# --- Pod Debugging ---
kubectl get pods -A                              # All pods across all namespaces
kubectl get pods -o wide                         # Pods with node and IP info
kubectl describe pod <pod-name>                  # Detailed pod info and events
kubectl logs <pod-name> -c <container>           # Logs for a specific container
kubectl logs <pod-name> --previous               # Logs from the previous crashed container
kubectl exec -it <pod-name> -- /bin/sh           # Shell into a running container
kubectl top pods                                 # CPU/memory usage per pod

# --- Node Debugging ---
kubectl get nodes -o wide                        # Node status with IPs and versions
kubectl describe node <node-name>                # Detailed node info
kubectl top nodes                                # CPU/memory usage per node
kubectl cordon <node-name>                       # Mark node as unschedulable
kubectl drain <node-name> --ignore-daemonsets     # Evict all pods from a node

# --- Service & Network Debugging ---
kubectl get svc -A                               # All services
kubectl get endpoints <svc-name>                 # Endpoints backing a service
kubectl port-forward svc/<svc-name> 8080:80      # Forward local port to service

# --- Events ---
kubectl get events --sort-by=.metadata.creationTimestamp   # Recent events sorted by time
kubectl get events -n <namespace> --field-selector type=Warning  # Warning events only

# --- Resource Usage & Quotas ---
kubectl describe resourcequota -n <namespace>    # Namespace resource quotas
kubectl describe limitrange -n <namespace>       # Default resource limits

# --- General ---
kubectl api-resources                            # All available resource types
kubectl explain <resource>                       # Documentation for a resource
kubectl diff -f <manifest.yaml>                  # Preview changes before applying
kubectl get all -n <namespace>                   # All resources in a namespace
```

---

## Quick Troubleshooting Flowchart

```
Pod not running?
├── Status: Pending
│   ├── Check: kubectl describe pod → Events
│   ├── Insufficient resources? → Scale cluster or reduce requests
│   ├── PVC unbound? → Fix StorageClass / PV
│   └── Node selector / affinity mismatch? → Fix labels
├── Status: CrashLoopBackOff
│   ├── Check: kubectl logs --previous
│   ├── App error? → Fix application
│   ├── Missing config? → Create ConfigMap/Secret
│   └── Probe failing? → Adjust probe settings
├── Status: ImagePullBackOff
│   ├── Wrong image/tag? → Fix image reference
│   └── Private registry? → Add imagePullSecrets
├── Status: CreateContainerConfigError
│   └── Missing ConfigMap/Secret? → Create the resource
└── Status: Running but not working
    ├── Check: kubectl logs
    ├── Check: kubectl describe pod → Readiness probe
    └── Check: kubectl exec → test connectivity
```

---

*Last updated: 2026-10-31*
