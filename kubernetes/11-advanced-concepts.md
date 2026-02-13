# 🚀 Chapter 11: Advanced Concepts

> **"Take your K8s skills to the next level!"**

---

## 🧩 Custom Resource Definitions (CRDs)

> **Extend Kubernetes with your own resource types**

```
┌────────────────────────────────────────────────────────────────┐
│                    CRD CONCEPT                                  │
│                                                                 │
│   Built-in Resources:                                          │
│   Pod, Service, Deployment, ConfigMap...                       │
│                                                                 │
│   Custom Resources (via CRD):                                  │
│   Database, Certificate, Backup, anything you define!         │
│                                                                 │
│   kubectl get databases   ← Your custom resource!              │
│   kubectl get certificates                                     │
└────────────────────────────────────────────────────────────────┘
```

### CRD Example

```yaml
# 1. Define the CRD
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
              version:
                type: string
              replicas:
                type: integer
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
    - db
---
# 2. Create instance of custom resource
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-postgres
spec:
  engine: postgresql
  version: "14.0"
  replicas: 3
```

### CRD Commands

```bash
kubectl get crd
kubectl get databases
kubectl get db  # using shortName
```

---

## 🤖 Operators

> **Automated operations for complex applications**

```
┌────────────────────────────────────────────────────────────────┐
│                    OPERATOR PATTERN                             │
│                                                                 │
│   ┌──────────────┐      ┌──────────────┐                       │
│   │   Operator   │      │   Custom     │                       │
│   │  (Controller)│ ◄──► │   Resource   │                       │
│   └──────┬───────┘      └──────────────┘                       │
│          │                                                      │
│          │ watches & manages                                   │
│          ▼                                                      │
│   ┌──────────────────────────────────────────────────────┐     │
│   │   Pods, Services, ConfigMaps, Secrets, etc.          │     │
│   │   (Whatever the operator needs to manage)            │     │
│   └──────────────────────────────────────────────────────┘     │
│                                                                 │
│   CRD + Controller = Operator                                  │
└────────────────────────────────────────────────────────────────┘
```

### Popular Operators

| Operator | Manages |
|----------|---------|
| **Prometheus Operator** | Monitoring |
| **Cert-Manager** | TLS certificates |
| **Postgres Operator** | PostgreSQL clusters |
| **Strimzi** | Apache Kafka |

---

## 📦 Helm

> **Package manager for Kubernetes (like apt/yum for K8s)**

```
┌────────────────────────────────────────────────────────────────┐
│                    HELM CONCEPTS                                │
│                                                                 │
│   Chart = Package (folder with templates + values)            │
│   Release = Instance of a chart installed in cluster          │
│   Repository = Collection of charts                            │
│                                                                 │
│   Chart Structure:                                              │
│   mychart/                                                      │
│   ├── Chart.yaml         # Metadata                            │
│   ├── values.yaml        # Default values                      │
│   ├── templates/         # K8s manifest templates              │
│   │   ├── deployment.yaml                                      │
│   │   ├── service.yaml                                         │
│   │   └── ingress.yaml                                         │
│   └── README.md                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Helm Commands

```bash
# Install
helm install my-release bitnami/nginx

# List releases
helm list

# Upgrade
helm upgrade my-release bitnami/nginx --set replicaCount=3

# Uninstall
helm uninstall my-release

# Search charts
helm search repo nginx
helm search hub nginx  # Artifact Hub
```

---

## 🔧 Init Containers

> **Run setup tasks before main containers start**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nc -z database 5432; do sleep 2; done']
  - name: init-config
    image: busybox
    command: ['sh', '-c', 'wget -O /data/config.json http://config-server/config']
    volumeMounts:
    - name: config-vol
      mountPath: /data
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config-vol
      mountPath: /app/config
  volumes:
  - name: config-vol
    emptyDir: {}
```

### Init Container Use Cases
- ✅ Wait for dependencies (database, services)
- ✅ Download configuration files
- ✅ Database migrations
- ✅ Clone Git repos
- ✅ Register with external service

---

## 🚗 Sidecar Containers

> **Helper containers that run alongside main app**

```
┌────────────────────────────────────────────────────────────────┐
│                    POD WITH SIDECAR                             │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                         POD                              │  │
│   │                                                          │  │
│   │   ┌─────────────┐      ┌─────────────┐                  │  │
│   │   │    Main     │      │   Sidecar   │                  │  │
│   │   │    App      │◄────►│   (logs)    │                  │  │
│   │   └──────┬──────┘      └──────┬──────┘                  │  │
│   │          │                    │                          │  │
│   │          └────────┬───────────┘                          │  │
│   │                   │                                      │  │
│   │            ┌──────┴──────┐                              │  │
│   │            │Shared Volume│                              │  │
│   │            └─────────────┘                              │  │
│   └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### Sidecar Use Cases
- 📊 Log shipping (Fluentd sidecar)
- 🔒 Proxy for service mesh (Envoy)
- 🔄 Config refresh
- 📈 Metrics collection

---

## 🕸️ Istio Service Mesh

> **Advanced traffic management, security, and observability**

```
┌────────────────────────────────────────────────────────────────┐
│                    SERVICE MESH                                 │
│                                                                 │
│   Without Mesh:                                                │
│   [App A] ──────────────────────► [App B]                     │
│                                                                 │
│   With Istio:                                                  │
│   [App A] ◄─► [Envoy Proxy] ──► [Envoy Proxy] ◄─► [App B]    │
│                     │                    │                     │
│                     └────────────────────┘                     │
│                        Control Plane                           │
│                     (Pilot, Citadel, Galley)                  │
│                                                                 │
│   Features:                                                    │
│   • Traffic management (canary, A/B testing)                   │
│   • mTLS between services                                      │
│   • Distributed tracing                                        │
│   • Circuit breakers                                           │
└────────────────────────────────────────────────────────────────┘
```

### Istio Components

| Component | Purpose |
|-----------|---------|
| **Envoy** | Sidecar proxy |
| **Pilot** | Traffic management |
| **Citadel** | Certificate management |
| **Galley** | Configuration management |

---

## 🔌 K8s API

> **All K8s operations go through the API**

```bash
# Direct API access
kubectl proxy  # Start proxy on localhost:8001

# API endpoints
curl http://localhost:8001/api/v1/pods
curl http://localhost:8001/api/v1/namespaces/default/pods
curl http://localhost:8001/apis/apps/v1/deployments

# API discovery
kubectl api-resources
kubectl api-versions
kubectl explain pods
kubectl explain pods.spec.containers
```

---

## 🧠 Memory Shortcuts

### Advanced Components: **"COHIS"**
```
C = CRD (Custom Resource Definition)
O = Operators (automate operations)
H = Helm (package manager)
I = Init containers (setup before main)
S = Sidecar (helper containers)
```

### Operator Pattern
```
CRD + Controller = Operator
(What you want) + (How to get it) = (Automation)
```

---

**Next: [12 - Monitoring & Logging](./12-monitoring-logging.md)** ➡️
