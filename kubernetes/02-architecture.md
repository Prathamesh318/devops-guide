# 🏗️ Chapter 2: Kubernetes Architecture Deep Dive

> **"Understanding K8s architecture is like understanding how a city works - you need to know who does what!"**

---

## 🎯 High-Level Overview

Kubernetes follows a **Master-Worker** architecture (also called **Control Plane - Data Plane**).

```
┌───────────────────────────────────────────────────────────────────────┐
│                      KUBERNETES CLUSTER                                │
│                                                                        │
│   ┌─────────────────────────────────────────────────────────────┐     │
│   │                    CONTROL PLANE (Master)                    │     │
│   │                    (The Brain 🧠)                           │     │
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────────┐    │     │
│   │   │   API   │ │  ETCD   │ │Scheduler│ │  Controller   │    │     │
│   │   │ Server  │ │         │ │         │ │   Manager     │    │     │
│   │   └─────────┘ └─────────┘ └─────────┘ └───────────────┘    │     │
│   └─────────────────────────────────────────────────────────────┘     │
│                                │                                       │
│                                │ (Communication)                       │
│                                ▼                                       │
│   ┌─────────────────────────────────────────────────────────────┐     │
│   │                    WORKER NODES (Data Plane)                 │     │
│   │                    (The Muscles 💪)                          │     │
│   │                                                              │     │
│   │   ┌─────────────────┐       ┌─────────────────┐             │     │
│   │   │   Worker Node 1  │       │   Worker Node 2  │             │     │
│   │   │ ┌──────┐┌──────┐│       │ ┌──────┐┌──────┐│             │     │
│   │   │ │ Pod  ││ Pod  ││       │ │ Pod  ││ Pod  ││             │     │
│   │   │ └──────┘└──────┘│       │ └──────┘└──────┘│             │     │
│   │   │    Kubelet      │       │    Kubelet      │             │     │
│   │   │    Kube-proxy   │       │    Kube-proxy   │             │     │
│   │   └─────────────────┘       └─────────────────┘             │     │
│   └─────────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Control Plane Components

The Control Plane is responsible for managing the cluster. Think of it as the **Management Office** of a company.

### 1. API Server (kube-apiserver) 🌐

**What it does:** The front door of Kubernetes - ALL communication goes through it.

```
┌──────────────────────────────────────────────────────────────┐
│                        API SERVER                             │
│                   "The Receptionist"                          │
│                                                               │
│    kubectl ────►  ┌─────────┐                                │
│    Dashboard ──►  │   API   │ ◄─── ETCD                      │
│    External ───►  │ Server  │ ◄─── Scheduler                 │
│    Tools ──────►  └─────────┘ ◄─── Controller                │
│                                                               │
│   • Validates requests                                        │
│   • Authenticates users                                       │
│   • REST API endpoint                                         │
└──────────────────────────────────────────────────────────────┘
```

**Real-world analogy:** Like a **Hotel Receptionist**
- All guests (users/tools) talk to receptionist first
- Receptionist validates reservations (authentication)
- Coordinates with housekeeping, kitchen (other components)

**Key Points:**
- ✅ Only component that talks to ETCD directly
- ✅ Exposes Kubernetes API (REST)
- ✅ Handles authentication & authorization
- ✅ Acts as gateway for CLI, UI, and API calls

---

### 2. ETCD 💾

**What it does:** Stores ALL cluster data - the source of truth.

```
┌──────────────────────────────────────────────────────────────┐
│                          ETCD                                 │
│                   "The Database Brain"                        │
│                                                               │
│   Key-Value Store that holds:                                 │
│   ┌──────────────────────────────────────────────────────┐   │
│   │ /registry/pods/default/nginx-pod                      │   │
│   │ /registry/services/default/my-service                 │   │
│   │ /registry/deployments/default/my-app                  │   │
│   │ /registry/secrets/default/db-password                 │   │
│   │ /registry/configmaps/default/app-config               │   │
│   └──────────────────────────────────────────────────────┘   │
│                                                               │
│   • Distributed & consistent                                  │
│   • Critical for cluster state                                │
│   • Needs backup!                                             │
└──────────────────────────────────────────────────────────────┘
```

**Real-world analogy:** Like a **Company's Record Room**
- Stores all employee records, policies, contracts
- Everyone refers to it for truth
- Must be protected and backed up

**Key Points:**
- ✅ Key-value store (like a dictionary)
- ✅ Stores cluster state, not application data
- ✅ Distributed for high availability
- ✅ Critical to backup regularly

**Memory tip:** **E**TCD = **E**verything is **S**tored here (ETCD sounds like "etcetera" - and everything!)

---

### 3. Scheduler (kube-scheduler) 📋

**What it does:** Assigns newly created pods to suitable nodes.

```
┌──────────────────────────────────────────────────────────────┐
│                       SCHEDULER                               │
│                   "The Assignment Manager"                    │
│                                                               │
│   New Pod Request ─────►  ┌──────────┐                       │
│                           │Scheduler │                       │
│                           │ Logic:   │                       │
│                           │          │                       │
│   "Where to place        │ 1. Filter│ ────► Node 1 ❌       │
│    this pod?"            │ 2. Score │ ────► Node 2 ✅       │
│                           │ 3. Bind  │ ────► Node 3 ❌       │
│                           └──────────┘                       │
│                                                               │
│   Considers: CPU, Memory, Affinity, Taints, etc.             │
└──────────────────────────────────────────────────────────────┘
```

**Real-world analogy:** Like a **Hotel Room Assigner**
- Guest arrives (new pod created)
- Checks which rooms are available (node resources)
- Considers guest preferences (affinity rules)
- Assigns best available room (binds pod to node)

**Scheduling Process:**
```
1. Filtering ──► Remove nodes that don't meet requirements
                 (not enough CPU, wrong zone, taints, etc.)

2. Scoring ───► Rank remaining nodes by desirability
                 (resource balance, affinity score, etc.)

3. Binding ───► Assign pod to highest-scoring node
```

**Key Points:**
- ✅ Doesn't actually run pods (just assigns)
- ✅ Considers resource requirements
- ✅ Respects constraints (taints, affinity)
- ✅ Runs continuous watch for unscheduled pods

---

### 4. Controller Manager (kube-controller-manager) 🎮

**What it does:** Runs controllers that monitor and maintain desired state.

```
┌──────────────────────────────────────────────────────────────────┐
│                     CONTROLLER MANAGER                            │
│                   "The Auto-Pilot System"                         │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │              CONTROLLERS (Control Loops)                 │    │
│   │                                                          │    │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│   │   │  Replication │  │    Node      │  │  Deployment  │  │    │
│   │   │  Controller  │  │  Controller  │  │  Controller  │  │    │
│   │   └──────────────┘  └──────────────┘  └──────────────┘  │    │
│   │                                                          │    │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│   │   │   Service    │  │  Endpoint    │  │    Job       │  │    │
│   │   │  Controller  │  │  Controller  │  │  Controller  │  │    │
│   │   └──────────────┘  └──────────────┘  └──────────────┘  │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│   Pattern: Watch → Compare → Act                                  │
│   "Current State" ≠ "Desired State" → Take Action!               │
└──────────────────────────────────────────────────────────────────┘
```

**Real-world analogy:** Like a **Thermostat + AC System**
- You set desired temperature (desired state)
- Thermostat constantly checks current temperature (watch)
- If current ≠ desired, AC turns on/off (reconcile)

**Common Controllers:**

| Controller | What it does |
|------------|-------------|
| **Node Controller** | Monitors node health, handles failures |
| **Replication Controller** | Maintains correct number of pod replicas |
| **Deployment Controller** | Manages deployment updates and rollbacks |
| **Service Controller** | Creates load balancers for services |
| **Job Controller** | Manages one-off job execution |

**Key Points:**
- ✅ Each controller is a separate loop
- ✅ Watches for changes via API Server
- ✅ Self-healing happens here!
- ✅ Runs in a single binary for efficiency

---

### 5. Cloud Controller Manager (optional) ☁️

**What it does:** Connects cluster to cloud provider's APIs.

```
┌─────────────────────────────────────────────────────────────┐
│              CLOUD CONTROLLER MANAGER                        │
│                                                              │
│   Only runs if using cloud provider (AWS, GCP, Azure)        │
│                                                              │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│   │    Node     │   │   Route     │   │   Service   │       │
│   │ Controller  │   │ Controller  │   │ Controller  │       │
│   │(Cloud VMs)  │   │(Cloud VPC)  │   │(Cloud LB)   │       │
│   └─────────────┘   └─────────────┘   └─────────────┘       │
│                                                              │
│   Talks to: AWS EC2, Azure VMs, GCP Compute, etc.           │
└─────────────────────────────────────────────────────────────┘
```

---

## 💪 Worker Node Components

Worker nodes run your actual applications. Think of them as the **Factory Floor** where work happens.

### 1. Kubelet 🤖

**What it does:** The agent running on each node that manages pods.

```
┌──────────────────────────────────────────────────────────────┐
│                        KUBELET                                │
│                   "The Node Manager"                          │
│                                                               │
│   On each worker node:                                        │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐    │
│   │  API Server says: "Run nginx pod on this node"      │    │
│   │                      │                               │    │
│   │                      ▼                               │    │
│   │  KUBELET receives instruction                        │    │
│   │                      │                               │    │
│   │                      ▼                               │    │
│   │  Kubelet tells Container Runtime: "Create container"│    │
│   │                      │                               │    │
│   │                      ▼                               │    │
│   │  Container runs! Kubelet monitors health.           │    │
│   └─────────────────────────────────────────────────────┘    │
│                                                               │
│   Reports back: "Pod is running, healthy, using X resources" │
└──────────────────────────────────────────────────────────────┘
```

**Real-world analogy:** Like a **Floor Supervisor in a Factory**
- Receives orders from management (API Server)
- Ensures workers (containers) are doing their jobs
- Reports status back to management
- Handles issues on the floor

**Key Points:**
- ✅ Runs on every node (including master in some setups)
- ✅ Communicates with API Server
- ✅ Uses Container Runtime (Docker, containerd, CRI-O)
- ✅ Handles health checks (probes)

---

### 2. Kube-proxy 🔀

**What it does:** Handles network rules and enables service communication.

```
┌──────────────────────────────────────────────────────────────────┐
│                        KUBE-PROXY                                 │
│                   "The Traffic Controller"                        │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │                                                          │    │
│   │   Request to Service ──► Kube-proxy ──► Backend Pod     │    │
│   │   (ClusterIP:80)                        (PodIP:8080)    │    │
│   │                                                          │    │
│   │   How?                                                   │    │
│   │   • Maintains network rules (iptables/IPVS)             │    │
│   │   • Forwards traffic to correct pods                     │    │
│   │   • Enables service discovery                            │    │
│   │                                                          │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│   Service: my-service (10.96.0.1:80)                             │
│              ↓                                                    │
│   kube-proxy routes to:                                          │
│   • Pod1 (10.244.0.5:8080)                                       │
│   • Pod2 (10.244.0.6:8080)                                       │
│   • Pod3 (10.244.0.7:8080)                                       │
└──────────────────────────────────────────────────────────────────┘
```

**Real-world analogy:** Like a **Mail Carrier in an Office Building**
- Knows where everyone sits (service-to-pod mapping)
- Routes mail to correct desks (forwards traffic)
- Updates route when people move (pod changes)

**Key Points:**
- ✅ Runs on every node
- ✅ Implements Kubernetes Services
- ✅ Uses iptables or IPVS for routing
- ✅ Enables load balancing across pods

---

### 3. Container Runtime 📦

**What it does:** Actually runs containers (the low-level engine).

```
┌──────────────────────────────────────────────────────────────┐
│                   CONTAINER RUNTIME                           │
│                  "The Container Engine"                       │
│                                                               │
│   Kubelet ──► CRI (Container Runtime Interface) ──► Runtime  │
│                                                               │
│   Supported Runtimes:                                         │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│   │ containerd  │  │   CRI-O     │  │   Docker*   │          │
│   │  (default)  │  │  (RedHat)   │  │(deprecated) │          │
│   └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                               │
│   * Docker as runtime deprecated in K8s 1.24+                 │
│     (Docker images still work!)                               │
└──────────────────────────────────────────────────────────────┘
```

**Key Points:**
- ✅ K8s is runtime-agnostic (uses CRI)
- ✅ containerd is most common now
- ✅ Docker images work with any runtime
- ✅ Responsible for pulling images, running containers

---

## 🔄 How It All Works Together

```
┌────────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT FLOW EXAMPLE                          │
│                                                                     │
│   1. User: kubectl apply -f deployment.yaml                        │
│                        │                                            │
│                        ▼                                            │
│   2. API Server: Validates & stores in ETCD                        │
│                        │                                            │
│                        ▼                                            │
│   3. Controller Manager: Sees new deployment                        │
│      → Creates ReplicaSet → Creates Pod specs                      │
│                        │                                            │
│                        ▼                                            │
│   4. Scheduler: Finds best node for each pod                       │
│      → Binds pod to node                                           │
│                        │                                            │
│                        ▼                                            │
│   5. Kubelet (on selected node): Sees pod assignment               │
│      → Tells container runtime to pull image                       │
│      → Starts container                                            │
│                        │                                            │
│                        ▼                                            │
│   6. Kube-proxy: Updates network rules                             │
│      → Pod is accessible via Service                               │
│                        │                                            │
│                        ▼                                            │
│   7. Done! 🎉 Application is running and accessible                │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ kubectl - The Command Line Tool

**kubectl** (pronounced "cube-control" or "cube-C-T-L") is your primary way to interact with Kubernetes.

```
┌──────────────────────────────────────────────────────────────┐
│                        KUBECTL                                │
│               "Your Remote Control for K8s"                   │
│                                                               │
│   You ──► kubectl ──► API Server ──► Cluster                 │
│                                                               │
│   Common Commands:                                            │
│   ┌────────────────────────────────────────────────────────┐ │
│   │ kubectl get pods          # List all pods              │ │
│   │ kubectl get nodes         # List all nodes             │ │
│   │ kubectl describe pod X    # Details of pod X           │ │
│   │ kubectl apply -f file.yaml # Create from YAML          │ │
│   │ kubectl delete pod X      # Delete pod X               │ │
│   │ kubectl logs pod-name     # View pod logs              │ │
│   │ kubectl exec -it pod bash # Enter pod shell            │ │
│   └────────────────────────────────────────────────────────┘ │
│                                                               │
│   Pro tip: alias k=kubectl                                   │
└──────────────────────────────────────────────────────────────┘
```

### kubectl Syntax Pattern

```
kubectl [verb] [resource] [name] [flags]

Examples:
kubectl  get     pods     nginx   -o wide
kubectl  delete  service  web-svc --namespace=prod
kubectl  apply   -f       app.yaml
```

---

## 🧠 Memory Shortcuts for Architecture

### Remember Control Plane with **"ACES"**
```
A = API Server     (Front door, all communication)
C = Controller     (Auto-healing, maintains state)
E = ETCD           (Database, stores everything)
S = Scheduler      (Assigns pods to nodes)
```

### Remember Node Components with **"KKC"**
```
K = Kubelet        (Node agent)
K = Kube-proxy     (Network rules)
C = Container Runtime (Runs containers)
```

### The Flow Mantra
```
"User talks to API,
API stores in ETCD,
Controller sees and acts,
Scheduler picks the node,
Kubelet runs the pod!"
```

---

## 🎨 Visual Architecture Diagram

```
                              ┌──────────────────────────────────┐
                              │         CONTROL PLANE            │
                              │    ┌─────┐ ┌─────┐ ┌─────────┐  │
      kubectl ────────────────┼──► │ API │◄│ETCD │ │Controller│  │
                              │    │     │ │     │ │ Manager  │  │
      Dashboard ──────────────┼──► │ SRV │ │     │ └─────────┘  │
                              │    └──┬──┘ └─────┘ ┌─────────┐  │
                              │       │            │Scheduler │  │
                              │       │            └─────────┘  │
                              └───────┼──────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
            ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
            │   NODE 1     │  │   NODE 2     │  │   NODE 3     │
            │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
            │  │Kubelet │  │  │  │Kubelet │  │  │  │Kubelet │  │
            │  ├────────┤  │  │  ├────────┤  │  │  ├────────┤  │
            │  │Kube-   │  │  │  │Kube-   │  │  │  │Kube-   │  │
            │  │proxy   │  │  │  │proxy   │  │  │  │proxy   │  │
            │  ├────────┤  │  │  ├────────┤  │  │  ├────────┤  │
            │  │Container│  │  │ │Container│  │  │ │Container│  │
            │  │Runtime │  │  │  │Runtime │  │  │  │Runtime │  │
            │  └────────┘  │  │  └────────┘  │  │  └────────┘  │
            │  ┌──┐ ┌──┐  │  │  ┌──┐ ┌──┐  │  │  ┌──┐ ┌──┐  │
            │  │P1│ │P2│  │  │  │P3│ │P4│  │  │  │P5│ │P6│  │
            │  └──┘ └──┘  │  │  └──┘ └──┘  │  │  └──┘ └──┘  │
            └──────────────┘  └──────────────┘  └──────────────┘
```

---

## ✅ Quick Quiz

1. Which component stores all cluster data?
2. Which component assigns pods to nodes?
3. What runs on every worker node? (3 components)
4. Which component is the "front door" of the cluster?
5. What does the Controller Manager do?

<details>
<summary>Click for Answers</summary>

1. **ETCD** - Key-value store
2. **Scheduler** (kube-scheduler)
3. **Kubelet, Kube-proxy, Container Runtime**
4. **API Server** (kube-apiserver)
5. Runs controllers that maintain desired state (self-healing)

</details>

---

**Next Chapter: [03 - Cluster Setup Guide](./03-cluster-setup.md)** ➡️
