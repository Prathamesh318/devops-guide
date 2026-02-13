# 📊 Chapter 9: Scheduling & Resource Management

> **"Control WHERE pods run and HOW MUCH resources they consume"**

---

## 🎯 Resource Requests and Limits

```
┌────────────────────────────────────────────────────────────────┐
│              REQUESTS vs LIMITS                                 │
│                                                                 │
│   REQUESTS = Guaranteed minimum resources                      │
│   • Used by scheduler for node selection                       │
│   • Pod won't start if node can't provide requests            │
│                                                                 │
│   LIMITS = Maximum allowed resources                           │
│   • Container killed if exceeds memory limit (OOMKilled)       │
│   • CPU throttled if exceeds CPU limit                         │
│                                                                 │
│   ┌──────────────────────────────────────────────────────┐    │
│   │         0         Request        Limit        Max    │    │
│   │         |------------|-------------|-----------|     │    │
│   │         │            │             │                 │    │
│   │         │ Guaranteed │ Burstable   │ Killed/throttle│    │
│   │         │   usage    │   zone      │                │    │
│   └──────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────┘
```

### Resource Spec in Pod

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
        memory: "128Mi"    # Guaranteed memory
        cpu: "250m"        # 0.25 CPU cores
      limits:
        memory: "256Mi"    # Max memory (OOMKilled if exceeded)
        cpu: "500m"        # Max CPU (throttled if exceeded)
```

### CPU Units

| Value | Meaning |
|-------|---------|
| `1` | 1 CPU core |
| `500m` | 0.5 cores (500 millicores) |
| `100m` | 0.1 cores |

### Memory Units

| Value | Meaning |
|-------|---------|
| `128Mi` | 128 Mebibytes |
| `1Gi` | 1 Gibibyte |
| `256M` | 256 Megabytes (decimal) |

---

## 📋 Resource Quotas

> **Limit total resources per namespace**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    # Compute resources
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    
    # Object counts
    pods: "20"
    services: "10"
    secrets: "20"
    persistentvolumeclaims: "5"
```

### Quota Commands

```bash
kubectl get resourcequota -n development
kubectl describe resourcequota compute-quota -n development
```

---

## 🏥 Probes (Health Checks)

> **Tell K8s how to check if your app is healthy**

### Probe Types

| Probe | Purpose | Action on Failure |
|-------|---------|-------------------|
| **Liveness** | Is container alive? | Restart container |
| **Readiness** | Is container ready for traffic? | Remove from service |
| **Startup** | Has container started? | Keep checking |

```
┌────────────────────────────────────────────────────────────────┐
│                    PROBE FLOW                                   │
│                                                                 │
│   Container Start                                               │
│        │                                                        │
│        ▼                                                        │
│   [Startup Probe] ──► Success ──► [Liveness + Readiness]       │
│        │                                                        │
│        └──► Failure (keep trying until success or give up)     │
│                                                                 │
│   Running:                                                      │
│   • Liveness fails → Container restarted                       │
│   • Readiness fails → Removed from Service endpoints           │
└────────────────────────────────────────────────────────────────┘
```

### Probe Methods

```yaml
# HTTP Probe
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

# TCP Probe
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 15
  periodSeconds: 10

# Exec Probe
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Complete Probe Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: healthy-pod
spec:
  containers:
  - name: app
    image: myapp
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /ready
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
```

---

## 🎨 Taints and Tolerations

> **Taints keep pods OFF nodes, Tolerations allow pods ON tainted nodes**

```
┌────────────────────────────────────────────────────────────────┐
│                TAINTS & TOLERATIONS                             │
│                                                                 │
│   Node with Taint: "gpu=true:NoSchedule"                       │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │   Node: gpu-node                                        │  │
│   │   Taint: gpu=true:NoSchedule                           │  │
│   │                                                         │  │
│   │   Pod without toleration → ❌ REJECTED                 │  │
│   │   Pod with toleration    → ✅ SCHEDULED                │  │
│   └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### Taint Effects

| Effect | Description |
|--------|-------------|
| **NoSchedule** | Don't schedule new pods here |
| **PreferNoSchedule** | Try to avoid, but can schedule |
| **NoExecute** | Evict existing pods + NoSchedule |

### Taint Commands

```bash
# Add taint to node
kubectl taint nodes node1 key=value:NoSchedule

# Remove taint
kubectl taint nodes node1 key=value:NoSchedule-

# View taints
kubectl describe node node1 | grep Taint
```

### Toleration in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: app
    image: gpu-app
```

---

## 📍 Node Affinity

> **Schedule pods to nodes matching specific criteria**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
  containers:
  - name: app
    image: nginx
```

### Affinity Types

| Type | Meaning |
|------|---------|
| **required...** | Must match (hard requirement) |
| **preferred...** | Try to match (soft preference) |

---

## 📈 HPA (Horizontal Pod Autoscaler)

> **Automatically scale pods based on metrics**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### HPA Commands

```bash
# Create HPA imperatively
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=50

# View HPA
kubectl get hpa
kubectl describe hpa app-hpa
```

---

## 📊 VPA (Vertical Pod Autoscaler)

> **Automatically adjust resource requests/limits**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, Recreate, Auto
```

---

## 🧠 Memory Shortcuts

### Probes: **"LSR"**
```
L = Liveness (alive? restart if not)
S = Startup (started? wait for it)
R = Readiness (ready? remove from service if not)
```

### Scheduling: **"TAN"**
```
T = Taints (repel pods from nodes)
A = Affinity (attract pods to nodes)
N = NodeSelector (simple node selection)
```

### Autoscaling: **"HV"**
```
H = HPA (Horizontal - more pods)
V = VPA (Vertical - bigger pods)
```

---

**Next: [10 - Security & RBAC](./10-security-rbac.md)** ➡️
