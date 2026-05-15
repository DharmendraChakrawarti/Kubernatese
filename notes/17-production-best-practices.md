# Module 17: Production Best Practices & Troubleshooting

> **Level:** 🔴 Advanced  
> **Objective:** Run Kubernetes in production — reliability, troubleshooting, and operational excellence.

---

## 17.1 Production Readiness Checklist

### Application Level
- [ ] Set **resource requests AND limits** for all containers
- [ ] Configure **liveness, readiness, and startup probes**
- [ ] Use **PodDisruptionBudgets** (PDB)
- [ ] Set `terminationGracePeriodSeconds` appropriately
- [ ] Use **anti-affinity** to spread replicas across nodes
- [ ] Configure **topology spread constraints**
- [ ] Run at least **3 replicas** for HA
- [ ] Use **preStop hooks** for graceful shutdown
- [ ] Set **imagePullPolicy** appropriately
- [ ] Use **specific image tags** (never `latest` in prod)

### Cluster Level
- [ ] **HA control plane** (3+ masters)
- [ ] **etcd backups** (automated, tested regularly)
- [ ] **RBAC** enabled with least-privilege
- [ ] **Network Policies** (default deny)
- [ ] **Pod Security Standards** (Restricted)
- [ ] **Resource Quotas** per namespace
- [ ] **LimitRanges** for defaults
- [ ] **Monitoring** (Prometheus + Grafana)
- [ ] **Logging** (centralized)
- [ ] **Cluster Autoscaler** configured
- [ ] **Audit logging** enabled
- [ ] Secrets **encrypted at rest**

---

## 17.2 Graceful Shutdown

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10 && kill -SIGTERM 1"]
      # Or HTTP
      preStop:
        httpGet:
          path: /shutdown
          port: 8080
```

**Shutdown sequence:**
```
1. Pod set to Terminating
2. Removed from Service endpoints
3. preStop hook runs
4. SIGTERM sent to container
5. Grace period countdown
6. SIGKILL if still running
```

---

## 17.3 Troubleshooting Guide

### Pod Issues

```bash
# Pod stuck in Pending
kubectl describe pod <pod-name>
# Check: insufficient resources, nodeSelector, taints, PVC binding

# Pod in CrashLoopBackOff
kubectl logs <pod-name> --previous    # Logs from crashed container
kubectl describe pod <pod-name>       # Check events

# Pod in ImagePullBackOff
kubectl describe pod <pod-name>       # Check image name/tag
# Fix: correct image, add imagePullSecrets

# Pod in Init:Error
kubectl logs <pod-name> -c <init-container-name>

# OOMKilled
kubectl describe pod <pod-name>       # Check Last State reason
# Fix: increase memory limits
```

### Common Pod States & Fixes

| State | Likely Cause | Fix |
|-------|-------------|-----|
| **Pending** | No node fits (resources/affinity) | Add nodes, fix selectors |
| **CrashLoopBackOff** | App crashes on start | Check logs --previous, fix app |
| **ImagePullBackOff** | Wrong image or no access | Fix image name, add registry secret |
| **OOMKilled** | Memory limit exceeded | Increase memory limit |
| **Evicted** | Node under pressure | Add resources, check quotas |
| **CreateContainerError** | ConfigMap/Secret missing | Create missing resources |
| **Init:Error** | Init container failed | Check init container logs |

### Service Issues

```bash
# Service not reaching pods
kubectl get endpoints <service-name>    # Should show pod IPs
kubectl get pods -l <selector-labels>   # Verify label match

# DNS not resolving
kubectl exec -it <pod> -- nslookup <service-name>
kubectl exec -it <pod> -- nslookup <service>.<namespace>.svc.cluster.local

# Check kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

### Node Issues

```bash
kubectl describe node <node-name>
kubectl get events --sort-by=.lastTimestamp

# Check conditions
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
READY:.status.conditions[-1].status,\
CPU:.status.capacity.cpu,\
MEMORY:.status.capacity.memory

# Debug node
kubectl debug node/<node-name> -it --image=busybox
```

---

## 17.4 Debugging Tools

```bash
# Ephemeral debug container
kubectl debug pod/<pod-name> -it --image=nicolaka/netshoot --target=app

# Temporary debug pod
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash

# Network debugging from inside cluster
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
# Inside: curl, dig, nslookup, tcpdump, iperf, ss, netstat

# Copy files from/to pod
kubectl cp <pod>:/path/to/file ./local-file
kubectl cp ./local-file <pod>:/path/to/file
```

---

## 17.5 Cluster Upgrade Strategy

```bash
# 1. Upgrade control plane (one minor version at a time!)
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.30.0-1.1
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.30.0

# 2. Upgrade kubelet on control plane
sudo apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
sudo systemctl restart kubelet

# 3. Upgrade worker nodes (one at a time)
kubectl drain <node> --ignore-daemonsets
# On the node:
sudo apt-get install -y kubeadm=1.30.0-1.1
sudo kubeadm upgrade node
sudo apt-get install -y kubelet=1.30.0-1.1
sudo systemctl restart kubelet
# From control plane:
kubectl uncordon <node>
```

**Rules:**
- Upgrade **one minor version at a time** (1.28 → 1.29, NOT 1.28 → 1.30)
- **Control plane first**, then workers
- **Drain nodes** before upgrading workers
- **Test in staging** before production

---

## 17.6 Disaster Recovery

### etcd Backup & Restore

```bash
# Automated backup (CronJob)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"         # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: bitnami/etcd:latest
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
          restartPolicy: OnFailure
```

### Velero (Full Cluster Backup)

```bash
# Install Velero
velero install --provider aws --bucket my-backup-bucket --secret-file ./credentials

# Backup
velero backup create my-backup --include-namespaces production
velero backup create full-backup --include-resources '*'

# Restore
velero restore create --from-backup my-backup

# Schedule
velero schedule create daily-backup --schedule="0 2 * * *"
```

---

## 17.7 Cost Optimization

1. **Right-size resources** — use VPA recommendations
2. **Use spot/preemptible instances** for non-critical workloads
3. **Cluster Autoscaler** — scale down idle nodes
4. **HPA** — scale pods based on actual demand
5. **Set resource quotas** — prevent over-provisioning
6. **Use shared namespaces** for dev/test environments
7. **Delete idle resources** — unused PVs, old images
8. **Multi-tenancy** — share clusters across teams

---

**Previous:** [← Module 16](./16-cicd.md) | **Next:** [Module 18 →](./18-cka-ckad-prep.md)
