# 📦 Chapter 4: Kubernetes Core Concepts

> **"Master these concepts and you'll understand 80% of Kubernetes!"**

---

## 🏢 Namespaces

> **Think of namespaces as different floors in an office building**

Namespaces are virtual clusters within a physical cluster providing:
- 🔒 **Isolation** between teams/projects
- 📛 **Name scoping** (same resource name in different namespaces)
- 🎛️ **Resource quotas** per namespace

### Default Namespaces

| Namespace | Purpose |
|-----------|---------|
| **default** | Where your resources go if you don't specify |
| **kube-system** | K8s internal components (DNS, proxy) |
| **kube-public** | Publicly accessible data |
| **kube-node-lease** | Node heartbeat data |

### Namespace Commands

```bash
# List namespaces
kubectl get ns

# Create namespace
kubectl create ns dev

# Use specific namespace
kubectl get pods -n production

# Set default namespace
kubectl config set-context --current --namespace=dev

# Delete namespace (⚠️ Deletes ALL resources!)
kubectl delete ns dev
```

### Memory Tip: **"DIPN"**
```
D = Default (your stuff)
I = kube-public (Info)
P = kube-system (Protected)
N = kube-node-lease (Node health)
```

---

## 🎪 Pods

> **The smallest deployable unit - like a pea pod holding peas!**

```
┌─────────────────────────────────────────────────────────────┐
│                           POD                                │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│   │ Container 1 │   │ Container 2 │   │ Container 3 │       │
│   │   (main)    │   │  (sidecar)  │   │  (logging)  │       │
│   └─────────────┘   └─────────────┘   └─────────────┘       │
│   Shared: IP, Volumes, Network namespace, Same node         │
└─────────────────────────────────────────────────────────────┘
```

### Pod Lifecycle States
- **Pending** - Waiting for scheduling/image pull
- **Running** - At least one container running
- **Succeeded** - All containers completed successfully
- **Failed** - At least one container failed
- **Unknown** - Cannot determine state

### Pod Commands

```bash
# Run pod
kubectl run nginx --image=nginx

# List pods
kubectl get pods -o wide

# Describe pod
kubectl describe pod nginx

# View logs
kubectl logs nginx -f

# Exec into pod
kubectl exec -it nginx -- /bin/bash

# Port forward
kubectl port-forward nginx 8080:80

# Delete pod
kubectl delete pod nginx
```

### Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:1.21
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
  restartPolicy: Always
```

### Memory: **"SEUSS"**
```
S = Smallest deployable unit
E = Ephemeral (temporary)
U = Unique IP per pod
S = Shared resources among containers
S = Single node only
```

---

## 🏷️ Labels and Selectors

> **Labels are "tags" to organize and find resources**

### Labels Commands

```bash
# Add label
kubectl label pod nginx env=production

# Show labels
kubectl get pods --show-labels

# Filter by label
kubectl get pods -l app=web
kubectl get pods -l 'env in (prod,staging)'

# Remove label
kubectl label pod nginx env-
```

### Selector Operators: **"INED"**
```
I = In (value in list)
N = NotIn (value not in list)
E = Exists (key exists)
D = DoesNotExist (key missing)
```

---

## 📝 Annotations

| Feature | Labels | Annotations |
|---------|--------|-------------|
| **Purpose** | Select/filter | Store metadata |
| **Queryable** | Yes | No |
| **Size** | 63 chars | 256KB |
| **Use** | Grouping | Tool configs |

---

## 📋 YAML Pattern: **"A-K-M-S"**

```yaml
apiVersion: v1           # A - API Version
kind: Pod                # K - Kind
metadata:                # M - Metadata
  name: my-resource
spec:                    # S - Spec
  containers: [...]
```

---

**Next: [05 - Workload Resources](./05-workloads.md)** ➡️
