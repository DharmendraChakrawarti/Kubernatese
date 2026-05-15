# Module 03: Setting Up Kubernetes

> **Level:** 🟢 Beginner  
> **Objective:** Install and configure Kubernetes using Minikube, kubeadm, and cloud services.

---

## 3.1 Local Development Options

### Minikube (Recommended for Learning)

```bash
# Install Minikube (Linux)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Install Minikube (Windows - PowerShell)
choco install minikube

# Install Minikube (macOS)
brew install minikube

# Start cluster
minikube start --driver=docker --cpus=2 --memory=4096

# Useful commands
minikube status
minikube dashboard          # Opens web UI
minikube service <name>     # Access a service
minikube ssh                # SSH into the node
minikube addons list        # List available addons
minikube addons enable metrics-server
minikube stop
minikube delete
```

### Kind (Kubernetes in Docker)
```bash
# Install
go install sigs.k8s.io/kind@latest
# OR
brew install kind

# Create cluster
kind create cluster --name my-cluster

# Multi-node cluster config
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kind delete cluster --name my-cluster
```

### k3d (k3s in Docker)
```bash
# Install
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Create cluster
k3d cluster create mycluster --agents 2 --port "8080:80@loadbalancer"
k3d cluster delete mycluster
```

---

## 3.2 Production Setup with kubeadm

### Prerequisites (All Nodes)

```bash
# 1. Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# 2. Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# 3. Set sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# 4. Install containerd
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
# Set SystemdCgroup = true in config.toml
sudo systemctl restart containerd

# 5. Install kubeadm, kubelet, kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize Control Plane (Master Node Only)

```bash
# Initialize cluster
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<MASTER_IP>

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI (Calico)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

### Join Worker Nodes

```bash
# Run the join command from kubeadm init output
sudo kubeadm join <MASTER_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# If token expired, generate new one
kubeadm token create --print-join-command
```

### Verify Cluster

```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

---

## 3.3 Cloud Managed Kubernetes

### AWS EKS
```bash
# Using eksctl
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 3

aws eks update-kubeconfig --name my-cluster --region us-east-1
```

### Google GKE
```bash
gcloud container clusters create my-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type e2-medium

gcloud container clusters get-credentials my-cluster --zone us-central1-a
```

### Azure AKS
```bash
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2 \
  --generate-ssh-keys

az aks get-credentials --resource-group myRG --name myAKSCluster
```

---

## 3.4 kubectl Installation

```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# macOS
brew install kubectl

# Windows
choco install kubernetes-cli

# Verify
kubectl version --client
```

### kubeconfig File

Location: `~/.kube/config`

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://192.168.1.100:6443
    certificate-authority-data: <base64-ca-cert>
  name: my-cluster
contexts:
- context:
    cluster: my-cluster
    user: admin
    namespace: default
  name: my-context
current-context: my-context
users:
- name: admin
  user:
    client-certificate-data: <base64-cert>
    client-key-data: <base64-key>
```

```bash
# Context management
kubectl config get-contexts
kubectl config use-context my-context
kubectl config set-context --current --namespace=dev
kubectl config view --minify
```

---

**Previous:** [← Module 02](./02-architecture.md) | **Next:** [Module 04 →](./04-kubectl-commands.md)
