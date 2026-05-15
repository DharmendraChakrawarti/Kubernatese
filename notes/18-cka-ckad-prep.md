# Module 18: CKA / CKAD Exam Preparation & Cheat Sheets

> **Level:** 🔴 Advanced  
> **Objective:** Quick reference for CKA/CKAD exams — imperative commands, common tasks, and tips.

---

## 18.1 Exam Overview

| Exam | Focus | Duration | Pass Score | Questions |
|------|-------|----------|-----------|-----------|
| **CKA** | Cluster Administration | 2 hours | 66% | 15-20 tasks |
| **CKAD** | Application Development | 2 hours | 66% | 15-20 tasks |
| **CKS** | Cluster Security | 2 hours | 67% | 15-20 tasks |

**Format:** Performance-based (real cluster, terminal-only), open book (kubernetes.io docs only).

---

## 18.2 Exam Environment Setup (First 2 Minutes)

```bash
# Set aliases
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kaf='kubectl apply -f'

# Enable autocompletion
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

# Set editor
export KUBE_EDITOR=nano    # or vim

# Quick dry-run alias
export do='--dry-run=client -o yaml'
# Usage: k run nginx --image=nginx $do > pod.yaml
```

---

## 18.3 Imperative Command Cheat Sheet

### Pods

```bash
# Create pod
k run nginx --image=nginx:1.25 --port=80
k run nginx --image=nginx $do > pod.yaml
k run nginx --image=nginx --labels="app=web,tier=frontend"
k run nginx --image=nginx --requests='cpu=100m,memory=128Mi' --limits='cpu=200m,memory=256Mi'
k run nginx --image=nginx --env="APP_ENV=prod" --env="LOG_LEVEL=info"
k run nginx --image=nginx --command -- sleep 3600

# Temporary pod for testing
k run test --rm -it --image=busybox -- sh
k run test --rm -it --image=nicolaka/netshoot -- bash
```

### Deployments

```bash
k create deployment app --image=nginx:1.25 --replicas=3
k create deployment app --image=nginx $do > deploy.yaml
k scale deployment app --replicas=5
k set image deployment/app nginx=nginx:1.26
k rollout status deployment/app
k rollout undo deployment/app
k rollout history deployment/app
```

### Services

```bash
k expose pod nginx --port=80 --type=ClusterIP --name=nginx-svc
k expose deployment app --port=80 --target-port=8080 --type=NodePort
k create service clusterip my-svc --tcp=80:8080
k create service nodeport my-svc --tcp=80:8080 --node-port=30080
```

### ConfigMaps & Secrets

```bash
k create configmap my-config --from-literal=key1=val1 --from-literal=key2=val2
k create configmap my-config --from-file=config.txt
k create configmap my-config --from-file=config-dir/

k create secret generic my-secret --from-literal=pass=s3cr3t
k create secret generic my-secret --from-file=cert.pem
k create secret tls my-tls --cert=tls.crt --key=tls.key
k create secret docker-registry reg --docker-server=hub.io --docker-username=u --docker-password=p
```

### Namespaces, ServiceAccounts, RBAC

```bash
k create namespace dev
k create serviceaccount app-sa -n dev

k create role pod-reader --verb=get,list,watch --resource=pods -n dev
k create rolebinding read-pods --role=pod-reader --serviceaccount=dev:app-sa -n dev

k create clusterrole node-reader --verb=get,list --resource=nodes
k create clusterrolebinding read-nodes --clusterrole=node-reader --user=jane

k auth can-i create pods --as jane
k auth can-i delete deployments --as system:serviceaccount:dev:app-sa -n dev
```

### Jobs & CronJobs

```bash
k create job my-job --image=busybox -- echo "Hello"
k create cronjob my-cron --image=busybox --schedule="*/5 * * * *" -- echo "Hello"
```

### Ingress

```bash
k create ingress my-ingress --rule="myapp.com/=my-svc:80" --class=nginx
k create ingress my-ingress --rule="myapp.com/api=api-svc:8080" --rule="myapp.com/web=web-svc:80"
```

---

## 18.4 Common Exam Tasks

### Task: Create Pod with Resource Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

### Task: Multi-Container Pod with Shared Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'while true; do echo $(date) >> /data/log.txt; sleep 5; done']
    volumeMounts:
    - name: shared
      mountPath: /data
  - name: reader
    image: busybox
    command: ['sh', '-c', 'tail -f /data/log.txt']
    volumeMounts:
    - name: shared
      mountPath: /data
  volumes:
  - name: shared
    emptyDir: {}
```

### Task: Create PV and PVC

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### Task: Create NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-80
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

### Task: Node Maintenance

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# Perform maintenance...
kubectl uncordon <node>
```

### Task: etcd Backup/Restore

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd.db \
  --data-dir=/var/lib/etcd-restored
# Then update etcd manifest to use new data-dir
```

### Task: Cluster Upgrade

```bash
# Control Plane
apt-get install -y kubeadm=1.30.0-1.1
kubeadm upgrade plan
kubeadm upgrade apply v1.30.0
apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
systemctl restart kubelet

# Worker (from control plane, then on worker)
kubectl drain <worker> --ignore-daemonsets
# On worker node:
apt-get install -y kubeadm=1.30.0-1.1
kubeadm upgrade node
apt-get install -y kubelet=1.30.0-1.1
systemctl restart kubelet
# From control plane:
kubectl uncordon <worker>
```

---

## 18.5 Quick Reference Tables

### API Resources

| Resource | Short | API Group | Namespaced |
|----------|-------|-----------|-----------|
| pods | po | v1 | ✅ |
| services | svc | v1 | ✅ |
| deployments | deploy | apps/v1 | ✅ |
| replicasets | rs | apps/v1 | ✅ |
| statefulsets | sts | apps/v1 | ✅ |
| daemonsets | ds | apps/v1 | ✅ |
| configmaps | cm | v1 | ✅ |
| secrets | - | v1 | ✅ |
| persistentvolumes | pv | v1 | ❌ |
| persistentvolumeclaims | pvc | v1 | ✅ |
| nodes | no | v1 | ❌ |
| namespaces | ns | v1 | ❌ |
| ingresses | ing | networking.k8s.io/v1 | ✅ |
| networkpolicies | netpol | networking.k8s.io/v1 | ✅ |
| jobs | - | batch/v1 | ✅ |
| cronjobs | cj | batch/v1 | ✅ |
| serviceaccounts | sa | v1 | ✅ |
| roles | - | rbac/v1 | ✅ |
| clusterroles | - | rbac/v1 | ❌ |

### Useful kubectl Shortcuts

```bash
k get po                    # pods
k get svc                   # services
k get deploy                # deployments
k get rs                    # replicasets
k get ds                    # daemonsets
k get sts                   # statefulsets
k get cm                    # configmaps
k get pv                    # persistentvolumes
k get pvc                   # persistentvolumeclaims
k get no                    # nodes
k get ns                    # namespaces
k get ing                   # ingresses
k get netpol                # networkpolicies
k get sa                    # serviceaccounts
k get cj                    # cronjobs
k get hpa                   # horizontalpodautoscalers
k get sc                    # storageclasses
k get ep                    # endpoints
```

---

## 18.6 Exam Tips

1. **Practice time management** — skip hard questions, come back later
2. **Use imperative commands** to save time (then edit YAML if needed)
3. **Use `$do` alias** (`--dry-run=client -o yaml`) to generate YAML quickly
4. **Bookmark important K8s docs pages** before the exam
5. **Use `kubectl explain`** instead of searching docs
6. **Copy-paste from docs** — don't type complex YAML from memory
7. **Check namespace** — many tasks specify a namespace, don't miss it
8. **Verify after each task** — `kubectl get`, `kubectl describe`
9. **Use `kubectl run --rm -it`** for quick testing
10. **Know vim/nano basics** — you'll edit YAML files in terminal

### Key Documentation Pages to Bookmark

- https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- https://kubernetes.io/docs/concepts/workloads/pods/
- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- https://kubernetes.io/docs/concepts/policy/network-policy/
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

---

## 18.7 CKA Domain Weights (2024)

| Domain | Weight |
|--------|--------|
| Cluster Architecture, Installation & Configuration | 25% |
| Workloads & Scheduling | 15% |
| Services & Networking | 20% |
| Storage | 10% |
| Troubleshooting | 30% |

## 18.8 CKAD Domain Weights (2024)

| Domain | Weight |
|--------|--------|
| Application Design and Build | 20% |
| Application Deployment | 20% |
| Application Observability and Maintenance | 15% |
| Application Environment, Configuration and Security | 25% |
| Services & Networking | 20% |

---

**Previous:** [← Module 17](./17-production-best-practices.md) | **Home:** [🏠 README](../README.md)
