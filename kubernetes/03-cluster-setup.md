# 🛠️ Chapter 3: Kubernetes Cluster Setup Guide

> **"Before you can drive, you need a car. Before you can use K8s, you need a cluster!"**

---

## 🎯 Overview: Ways to Create a Kubernetes Cluster

```
┌────────────────────────────────────────────────────────────────────┐
│                    K8s CLUSTER OPTIONS                              │
│                                                                     │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│   │   LOCAL DEV     │  │   PRODUCTION    │  │     CLOUD       │   │
│   │                 │  │                 │  │    MANAGED      │   │
│   │  • KIND         │  │  • Kubeadm      │  │  • AWS EKS      │   │
│   │  • Minikube     │  │  • K3s          │  │  • Azure AKS    │   │
│   │  • Docker       │  │  • RKE          │  │  • Google GKE   │   │
│   │    Desktop      │  │  • Kubespray    │  │  • DigitalOcean │   │
│   │                 │  │                 │  │                 │   │
│   │  Best for:      │  │  Best for:      │  │  Best for:      │   │
│   │  Learning       │  │  Self-hosted    │  │  Production     │   │
│   │  Testing        │  │  On-premise     │  │  Enterprise     │   │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Prerequisites for All Setups

### System Requirements

| Tool | Minimum RAM | Recommended RAM | CPU Cores |
|------|------------|-----------------|-----------|
| KIND | 2 GB | 4 GB | 2+ |
| Minikube | 2 GB | 4 GB | 2+ |
| Kubeadm (Master) | 2 GB | 4 GB | 2+ |
| Kubeadm (Worker) | 1 GB | 2 GB | 1+ |

### Software Prerequisites

```bash
# 1. Docker (needed for KIND, container runtime)
docker --version

# 2. kubectl (K8s CLI - needed for all)
kubectl version --client

# 3. Git (for downloading examples)
git --version
```

---

## 🐳 KIND (Kubernetes IN Docker)

> **Best for:** Local development, CI/CD pipelines, quick testing

### What is KIND?

```
┌────────────────────────────────────────────────────────────────────┐
│                           KIND                                      │
│              "Kubernetes IN Docker"                                 │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    Your Computer                             │  │
│   │   ┌───────────────────────────────────────────────────────┐ │  │
│   │   │                    Docker                              │ │  │
│   │   │   ┌─────────────────────────────────────────────────┐ │ │  │
│   │   │   │        KIND Cluster                              │ │ │  │
│   │   │   │   ┌─────────────────────────────────────┐       │ │ │  │
│   │   │   │   │  Control Plane (Docker Container)   │       │ │ │  │
│   │   │   │   └─────────────────────────────────────┘       │ │ │  │
│   │   │   │   ┌─────────────┐ ┌─────────────┐               │ │ │  │
│   │   │   │   │ Worker Node │ │ Worker Node │               │ │ │  │
│   │   │   │   │ (Container) │ │ (Container) │               │ │ │  │
│   │   │   │   └─────────────┘ └─────────────┘               │ │ │  │
│   │   │   └─────────────────────────────────────────────────┘ │ │  │
│   │   └───────────────────────────────────────────────────────┘ │  │
│   └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘

Each K8s node = Docker container running K8s components!
```

### Why KIND?

| Advantage | Description |
|-----------|-------------|
| ⚡ **Fast** | Clusters start in seconds |
| 🪶 **Lightweight** | Uses Docker, minimal resources |
| 🔄 **Reproducible** | Config files define exact setup |
| 🧪 **CI/CD friendly** | Perfect for automated testing |
| 💵 **Free** | No cloud costs |

---

### KIND Installation

#### Windows (PowerShell - Run as Admin)

```powershell
# Option 1: Using Chocolatey
choco install kind -y

# Option 2: Download directly
# Go to: https://github.com/kubernetes-sigs/kind/releases
# Download kind-windows-amd64 and rename to kind.exe
# Add to PATH
```

#### Linux / macOS

```bash
# Linux (AMD64)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# macOS
brew install kind
```

#### Verify Installation

```bash
kind version
# Output: kind v0.20.0 go1.20.4 linux/amd64
```

---

### KIND Cluster Creation

#### Simple Single-Node Cluster

```bash
# Create cluster with default name "kind"
kind create cluster

# Create cluster with custom name
kind create cluster --name my-cluster

# Check cluster
kubectl get nodes
```

#### Multi-Node Cluster (Config File)

Create `kind-config.yaml`:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  # Control Plane Node
  - role: control-plane
    kubeadmConfigPatches:
    - |
      kind: InitConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "ingress-ready=true"
    extraPortMappings:
    - containerPort: 80
      hostPort: 80
      protocol: TCP
    - containerPort: 443
      hostPort: 443
      protocol: TCP
  # Worker Nodes
  - role: worker
  - role: worker
  - role: worker
```

```bash
# Create from config
kind create cluster --config kind-config.yaml --name dev-cluster

# List clusters
kind get clusters

# Get kubeconfig
kind get kubeconfig --name dev-cluster

# Delete cluster
kind delete cluster --name dev-cluster
```

---

### KIND Common Commands

```bash
┌───────────────────────────────────────────────────────────────────┐
│                    KIND CHEAT SHEET                                │
├───────────────────────────────────────────────────────────────────┤
│  kind create cluster                    # Create default cluster  │
│  kind create cluster --name NAME        # Create named cluster    │
│  kind create cluster --config file.yaml # Create from config      │
│  kind get clusters                      # List all clusters       │
│  kind get kubeconfig --name NAME        # Get cluster config      │
│  kind delete cluster                    # Delete default cluster  │
│  kind delete cluster --name NAME        # Delete named cluster    │
│  kind load docker-image IMAGE           # Load local image        │
└───────────────────────────────────────────────────────────────────┘
```

---

## 🎮 Minikube

> **Best for:** Beginners, feature-rich local development, VM-based isolation

### What is Minikube?

```
┌────────────────────────────────────────────────────────────────────┐
│                         MINIKUBE                                    │
│              "Mini Kubernetes - Single Node Cluster"                │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    Your Computer                             │  │
│   │                                                              │  │
│   │   ┌───────────────────────────────────────────────────────┐ │  │
│   │   │        Minikube VM or Container                        │ │  │
│   │   │   ┌─────────────────────────────────────────────┐     │ │  │
│   │   │   │         Single Node Cluster                  │     │ │  │
│   │   │   │   ┌─────────────────┐   ┌─────────────────┐ │     │ │  │
│   │   │   │   │ Control Plane   │ + │  Worker Node    │ │     │ │  │
│   │   │   │   │  (same node)    │   │  (same node)    │ │     │ │  │
│   │   │   │   └─────────────────┘   └─────────────────┘ │     │ │  │
│   │   │   └─────────────────────────────────────────────┘     │ │  │
│   │   └───────────────────────────────────────────────────────┘ │  │
│   └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘

Runs control plane + worker on same node (VM or container)
```

### KIND vs Minikube

| Feature | KIND | Minikube |
|---------|------|----------|
| **Nodes** | Multi-node in containers | Single node (multi-node experimental) |
| **Runtime** | Docker only | Docker, VM, Podman |
| **Speed** | Faster | Slightly slower |
| **Features** | Basic | Dashboard, addons, tunnels |
| **Use case** | CI/CD, quick tests | Learning, full features |

---

### Minikube Installation

#### Windows (PowerShell - Run as Admin)

```powershell
# Using Chocolatey
choco install minikube -y

# Or download installer from:
# https://minikube.sigs.k8s.io/docs/start/
```

#### Linux

```bash
# Download and install
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

#### macOS

```bash
brew install minikube
```

#### Verify Installation

```bash
minikube version
# minikube version: v1.31.2
```

---

### Minikube Cluster Creation

```bash
# Start with defaults (auto-selects driver)
minikube start

# Start with specific driver
minikube start --driver=docker         # Use Docker
minikube start --driver=virtualbox     # Use VirtualBox VM
minikube start --driver=hyperv         # Windows Hyper-V

# Start with custom resources
minikube start --cpus=4 --memory=8192

# Start specific K8s version
minikube start --kubernetes-version=v1.28.0

# Check status
minikube status
```

---

### Minikube Common Commands

```bash
┌───────────────────────────────────────────────────────────────────┐
│                   MINIKUBE CHEAT SHEET                             │
├───────────────────────────────────────────────────────────────────┤
│  Lifecycle:                                                        │
│  minikube start                # Start cluster                    │
│  minikube stop                 # Stop cluster                     │
│  minikube delete               # Delete cluster                   │
│  minikube pause                # Pause without stopping           │
│  minikube unpause              # Resume paused cluster            │
│                                                                    │
│  Information:                                                      │
│  minikube status               # Check cluster status             │
│  minikube ip                   # Get cluster IP                   │
│  minikube logs                 # View logs                        │
│                                                                    │
│  Features:                                                         │
│  minikube dashboard            # Open K8s dashboard              │
│  minikube addons list          # List available addons           │
│  minikube addons enable NAME   # Enable addon                    │
│  minikube tunnel               # Expose LoadBalancer services    │
│  minikube ssh                  # SSH into node                   │
│                                                                    │
│  Multi-node (experimental):                                        │
│  minikube start --nodes 3      # Start multi-node                │
│  minikube node add             # Add a node                      │
└───────────────────────────────────────────────────────────────────┘
```

### Minikube Addons (Built-in Features)

```bash
# List addons
minikube addons list

# Popular addons to enable:
minikube addons enable dashboard      # K8s Dashboard UI
minikube addons enable metrics-server # Resource metrics
minikube addons enable ingress        # Nginx Ingress Controller

# Open dashboard
minikube dashboard
```

---

## 🔧 Kubeadm (Production Setup)

> **Best for:** Production clusters, self-hosted infrastructure, learning real setup

### What is Kubeadm?

```
┌────────────────────────────────────────────────────────────────────┐
│                         KUBEADM                                     │
│            "Official K8s Cluster Bootstrapping Tool"                │
│                                                                     │
│   kubeadm does:                                                     │
│   ✅ Initialize control plane                                       │
│   ✅ Join worker nodes                                              │
│   ✅ Upgrade clusters                                               │
│                                                                     │
│   kubeadm does NOT:                                                 │
│   ❌ Provision machines (you need VMs/servers)                      │
│   ❌ Install container runtime                                      │
│   ❌ Install kubectl or kubelet                                     │
│   ❌ Configure networking (you add CNI)                             │
└────────────────────────────────────────────────────────────────────┘
```

### Architecture for Kubeadm Setup

```
┌─────────────────────────────────────────────────────────────────────┐
│                  KUBEADM CLUSTER SETUP                               │
│                                                                      │
│    ┌─────────────────────────────────────────────────────────────┐  │
│    │              CONTROL PLANE (Master Node)                     │  │
│    │                                                              │  │
│    │   ┌──────────────────────────────────────────────────────┐  │  │
│    │   │  OS: Ubuntu 22.04 / CentOS / Amazon Linux            │  │  │
│    │   │  Container Runtime: containerd                        │  │  │
│    │   │  kubeadm, kubelet, kubectl installed                 │  │  │
│    │   │                                                       │  │  │
│    │   │  After kubeadm init:                                  │  │  │
│    │   │  • API Server, ETCD, Controller, Scheduler running   │  │  │
│    │   └──────────────────────────────────────────────────────┘  │  │
│    └─────────────────────────────────────────────────────────────┘  │
│                           │                                          │
│                           │ kubeadm join                             │
│                           ▼                                          │
│    ┌───────────────────────────┐   ┌───────────────────────────┐    │
│    │      WORKER NODE 1        │   │      WORKER NODE 2        │    │
│    │  Container Runtime        │   │  Container Runtime        │    │
│    │  kubelet                  │   │  kubelet                  │    │
│    │  kube-proxy               │   │  kube-proxy               │    │
│    └───────────────────────────┘   └───────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

### AWS Setup for Kubeadm Practice

#### Step 1: Create EC2 Instances

```
┌────────────────────────────────────────────────────────────────────┐
│                AWS EC2 INSTANCES NEEDED                             │
│                                                                     │
│   ┌──────────────────┬─────────────────┬─────────────────┐         │
│   │    Instance      │   Type          │   Purpose       │         │
│   ├──────────────────┼─────────────────┼─────────────────┤         │
│   │ k8s-master       │ t2.medium       │ Control Plane   │         │
│   │ k8s-worker-1     │ t2.micro/medium │ Worker Node     │         │
│   │ k8s-worker-2     │ t2.micro/medium │ Worker Node     │         │
│   └──────────────────┴─────────────────┴─────────────────┘         │
│                                                                     │
│   Common Config:                                                    │
│   • AMI: Ubuntu 22.04 LTS                                          │
│   • Security Group: Allow ports 22, 6443, 10250-10260, 30000-32767│
│   • Key Pair: Your SSH key                                         │
└────────────────────────────────────────────────────────────────────┘
```

#### Security Group Rules

```
┌───────────────────────────────────────────────────────────────────┐
│  SECURITY GROUP RULES FOR K8S CLUSTER                             │
├───────────────────────────────────────────────────────────────────┤
│  CONTROL PLANE:                                                    │
│  Port 6443       - Kubernetes API Server                          │
│  Port 2379-2380  - ETCD                                           │
│  Port 10250      - Kubelet API                                    │
│  Port 10259      - kube-scheduler                                 │
│  Port 10257      - kube-controller-manager                        │
│                                                                    │
│  WORKER NODES:                                                     │
│  Port 10250      - Kubelet API                                    │
│  Port 30000-32767 - NodePort Services                             │
│                                                                    │
│  ALL NODES:                                                        │
│  Port 22         - SSH access                                     │
└───────────────────────────────────────────────────────────────────┘
```

---

### Kubeadm Installation (All Nodes)

Run these commands on **ALL nodes** (master and workers):

```bash
#!/bin/bash
# Step 1: Disable swap (K8s requirement)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Step 2: Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Step 3: Set sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Step 4: Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Step 5: Install kubeadm, kubelet, kubectl
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Add K8s apt key and repo
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

### Initialize Control Plane (Master Node Only)

```bash
# On Master Node ONLY
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Save the output! It contains the join command for workers!
```

After successful init, configure kubectl:

```bash
# Configure kubectl for current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install CNI (Network Plugin)

```bash
# Install Flannel CNI (simple, works well)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# OR install Calico (more features)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Verify control plane is ready
kubectl get nodes
# NAME         STATUS   ROLES           AGE   VERSION
# k8s-master   Ready    control-plane   5m    v1.29.0
```

---

### Join Worker Nodes

On **each worker node**, run the join command from kubeadm init output:

```bash
# Example join command (yours will be different!)
sudo kubeadm join 192.168.1.100:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1234567890abcdef...

# If you lost the token, generate new one on master:
kubeadm token create --print-join-command
```

### Verify Cluster

```bash
# On master node
kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# k8s-master     Ready    control-plane   10m   v1.29.0
# k8s-worker-1   Ready    <none>          5m    v1.29.0
# k8s-worker-2   Ready    <none>          3m    v1.29.0

# Test with a pod
kubectl run nginx --image=nginx
kubectl get pods
```

---

### Kubeadm Commands Summary

```bash
┌───────────────────────────────────────────────────────────────────┐
│                   KUBEADM CHEAT SHEET                              │
├───────────────────────────────────────────────────────────────────┤
│  Cluster Lifecycle:                                                │
│  kubeadm init                    # Initialize control plane       │
│  kubeadm join <master>:<port>    # Join worker to cluster        │
│  kubeadm reset                   # Reset node (clean up)         │
│                                                                    │
│  Tokens:                                                           │
│  kubeadm token list              # List join tokens              │
│  kubeadm token create            # Create new token              │
│  kubeadm token create --print-join-command  # Generate join cmd  │
│                                                                    │
│  Upgrades:                                                         │
│  kubeadm upgrade plan            # Check available upgrades      │
│  kubeadm upgrade apply v1.29.0   # Upgrade control plane         │
│  kubeadm upgrade node            # Upgrade worker node           │
│                                                                    │
│  Config:                                                           │
│  kubeadm config print init-defaults    # View default config     │
│  kubeadm config images list            # List required images    │
│  kubeadm config images pull            # Pre-pull images         │
└───────────────────────────────────────────────────────────────────┘
```

---

## 📊 Comparison: Which to Choose?

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CLUSTER SETUP DECISION TREE                        │
│                                                                       │
│                        What's your goal?                              │
│                              │                                        │
│           ┌──────────────────┼──────────────────┐                    │
│           │                  │                  │                    │
│           ▼                  ▼                  ▼                    │
│      Learning?         Quick tests?        Production?              │
│           │                  │                  │                    │
│           ▼                  ▼                  ▼                    │
│      ┌─────────┐      ┌─────────┐      ┌──────────────┐            │
│      │MINIKUBE │      │  KIND   │      │   Cloud or   │            │
│      │Full     │      │Fast     │      │   Kubeadm    │            │
│      │features │      │light    │      │              │            │
│      └─────────┘      └─────────┘      └──────────────┘            │
└──────────────────────────────────────────────────────────────────────┘
```

| Criteria | KIND | Minikube | Kubeadm | Cloud (EKS/AKS/GKE) |
|----------|------|----------|---------|---------------------|
| **Setup Time** | 1 min | 2-3 min | 30+ min | 10-20 min |
| **Multi-node** | ✅ Easy | ⚠️ Experimental | ✅ Yes | ✅ Yes |
| **Resources** | Low | Medium | High | Depends |
| **Production** | ❌ | ❌ | ✅ | ✅ |
| **Cost** | Free | Free | Your infra | Pay cloud |
| **Best for** | CI/CD | Learning | Self-hosted | Enterprise |

---

## 🧠 Memory Shortcuts

### Remember Setup Options: **"KMK-C"**
```
K = KIND (Kubernetes IN Docker - fastest)
M = Minikube (feature-rich local)
K = Kubeadm (production self-hosted)
C = Cloud managed (EKS, AKS, GKE)
```

### Kubeadm Flow: **"I-J-V"**
```
I = Init (master node - kubeadm init)
J = Join (workers - kubeadm join)
V = Verify (kubectl get nodes)
```

---

## ✅ Quick Quiz

1. Which tool runs K8s nodes as Docker containers?
2. Which tool is best for beginners with a built-in dashboard?
3. What command initializes a control plane with kubeadm?
4. Do you need to install CNI separately when using Minikube?

<details>
<summary>Click for Answers</summary>

1. **KIND** (Kubernetes IN Docker)
2. **Minikube** (use `minikube dashboard`)
3. `kubeadm init`
4. **No** - Minikube handles it automatically (but yes for kubeadm)

</details>

---

**Next Chapter: [04 - Core Concepts](./04-core-concepts.md)** ➡️
