# 🎯 Chapter 16: Kubernetes Interview Questions & Production Decision-Making Guide

> **"Knowing Kubernetes is theory. Deciding REPLICAS, RESOURCES, and STRATEGIES in production — that's engineering."**

---

## 📌 Part 1: Most Asked Interview Questions (With Real Answers)

---

### Q1: What happens when you run `kubectl apply -f deployment.yaml`?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     FULL LIFECYCLE OF kubectl apply                     │
│                                                                         │
│  You (kubectl) ──► API Server ──► ETCD (store desired state)           │
│                        │                                                │
│                        ▼                                                │
│              Controller Manager                                         │
│              (sees: "desired=3, current=0")                             │
│                        │                                                │
│                        ▼                                                │
│              Creates ReplicaSet ──► Creates 3 Pods (Pending)           │
│                                                                         │
│              Scheduler picks nodes for each pod                         │
│                        │                                                │
│                        ▼                                                │
│              Kubelet on each node pulls image & starts container        │
│                        │                                                │
│                        ▼                                                │
│              Pod status: Pending → ContainerCreating → Running          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Interview Pro Tip:** Don't just say "it creates pods." Walk through API Server → ETCD → Controller → Scheduler → Kubelet flow.

---

### Q2: What's the difference between a Pod restart and a Pod re-creation?

| Aspect | Restart | Re-creation |
|--------|---------|-------------|
| **Who does it** | Kubelet (same node) | Controller (any node) |
| **When** | Liveness probe fails, OOMKilled | Node dies, deployment update |
| **Pod IP** | Same | New IP |
| **Local data** | Preserved (emptyDir survives) | Lost (unless PVC used) |
| **Pod Name** | Same | New random suffix |

**Real scenario:** Your app crashes due to memory leak → Kubelet restarts it (same node, same pod name). If the NODE dies → Controller creates a brand new pod on a different node.

---

### Q3: What's the difference between `kubectl create` and `kubectl apply`?

| Feature | `create` | `apply` |
|---------|----------|---------|
| First time | ✅ Works | ✅ Works |
| Second time (same resource) | ❌ Error: already exists | ✅ Updates it |
| Approach | Imperative (do exactly this) | Declarative (make it look like this) |
| Production use | Rarely | Almost always |

**Rule:** In production, ALWAYS use `apply`. It's idempotent — you can run it 100 times safely.

---

### Q4: How does Kubernetes self-heal?

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SELF-HEALING MECHANISM                            │
│                                                                      │
│   Desired State (ETCD): 3 replicas                                  │
│   Current State: 2 running (1 crashed)                              │
│                                                                      │
│   Controller Manager:                                                │
│   "Hmm, desired=3, actual=2 → I need to create 1 more pod"         │
│                                                                      │
│   Scenarios:                                                         │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │ Container crash  → Kubelet restarts (restartPolicy: Always) │  │
│   │ Pod deleted      → Controller creates new pod               │  │
│   │ Node dies        → Pods rescheduled to healthy nodes        │  │
│   │ Health check fail→ Liveness: restart | Readiness: remove    │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   KEY: K8s doesn't "fix" — it REPLACES.                             │
│   It deletes the broken and creates a fresh one.                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Q5: What happens when a Node goes down?

**Timeline:**
1. **0–40 seconds:** Kubelet stops sending heartbeats to API Server
2. **40 seconds:** Node Controller marks node as `NotReady`
3. **5 minutes (default `pod-eviction-timeout`):** Pods on dead node marked for eviction
4. **Controller** creates replacement pods on healthy nodes
5. **Services** automatically route traffic away from the dead node

**Interview follow-up:** "What if it was a StatefulSet?"
- StatefulSet waits LONGER because it needs to be sure the old pod is truly dead (no split-brain for databases)
- You may need to manually delete the pod or force-delete it

---

### Q6: Kubernetes Networking — How do Pods communicate?

```
┌──────────────────────────────────────────────────────────────────────┐
│                     NETWORKING RULES                                  │
│                                                                       │
│  Rule 1: Every Pod gets its own unique IP                            │
│  Rule 2: All Pods can communicate with all other Pods (no NAT)       │
│  Rule 3: All Nodes can communicate with all Pods (no NAT)            │
│                                                                       │
│  ┌──────────────┐        ┌──────────────┐                            │
│  │   Node 1     │        │   Node 2     │                            │
│  │  ┌────────┐  │        │  ┌────────┐  │                            │
│  │  │Pod A   │──┼── CNI ─┼──│Pod B   │  │                            │
│  │  │10.1.1.5│  │ Plugin │  │10.1.2.8│  │                            │
│  │  └────────┘  │        │  └────────┘  │                            │
│  └──────────────┘        └──────────────┘                            │
│                                                                       │
│  Communication types:                                                 │
│  • Pod-to-Pod (same node)  → Virtual bridge (cbr0)                   │
│  • Pod-to-Pod (diff node)  → CNI plugin (Calico, Flannel, Cilium)    │
│  • Pod-to-Service          → kube-proxy (iptables/IPVS rules)        │
│  • External-to-Service     → Ingress / LoadBalancer / NodePort       │
└──────────────────────────────────────────────────────────────────────┘
```

---

### Q7: ClusterIP vs NodePort vs LoadBalancer vs Ingress — When to use what?

| Type | Access From | Use Case | Cost |
|------|-------------|----------|------|
| **ClusterIP** | Inside cluster only | Microservice-to-microservice | Free |
| **NodePort** | Outside via `NodeIP:Port` | Dev/testing, on-prem | Free (exposes port 30000-32767) |
| **LoadBalancer** | Outside via cloud LB | Production single-service | $$$ (one LB per service) |
| **Ingress** | Outside via hostname/path | Production multi-service | $ (one LB for all services) |

**Production decision:**
```
Do you need external access?
├── NO → ClusterIP
└── YES
    ├── Single service? → LoadBalancer (small apps)
    └── Multiple services? → Ingress (ALWAYS prefer this)
         └── Use: nginx-ingress, Traefik, or cloud-native ALB ingress
```

**Real-world cost example:**
- 10 microservices with LoadBalancer each → 10 × $18/month = **$180/month** on AWS
- 10 microservices with 1 Ingress → 1 × $18/month = **$18/month** on AWS

---

### Q8: What is a Service Mesh and when do you need it?

**You DON'T need it when:**
- You have < 10 microservices
- Simple request routing
- Basic health checks are enough

**You NEED it when:**
- Mutual TLS between all services (zero-trust)
- Advanced traffic splitting (canary 5% traffic to v2)
- Distributed tracing across services
- Circuit breaking and retry policies
- You have 50+ microservices

**Popular options:** Istio (feature-rich, heavy), Linkerd (lightweight), Consul Connect

---

## 📌 Part 2: How Many Replicas to Keep and WHY

---

### The Replica Decision Framework

```
┌──────────────────────────────────────────────────────────────────────┐
│                 HOW MANY REPLICAS DO I NEED?                         │
│                                                                       │
│  Ask yourself these questions:                                       │
│                                                                       │
│  1. Can my app handle being down for 30 seconds?                     │
│     └── YES → 1 replica is OK (dev/staging)                         │
│     └── NO  → Minimum 2 replicas                                    │
│                                                                       │
│  2. How much traffic does it handle?                                 │
│     └── Each pod handles ~500 RPS → for 2000 RPS, need 4 pods       │
│                                                                       │
│  3. Am I doing rolling updates?                                      │
│     └── YES → Need at least 2 (so 1 stays alive during update)      │
│                                                                       │
│  4. Do I need high-availability across zones?                        │
│     └── YES → Minimum 3 (one per availability zone)                 │
│                                                                       │
│  5. Is this a critical service (payments, auth)?                     │
│     └── YES → Minimum 3, with PodDisruptionBudget                   │
└──────────────────────────────────────────────────────────────────────┘
```

### Real-World Replica Guidelines

| Environment | Service Type | Replicas | Why |
|-------------|-------------|----------|-----|
| **Dev/Local** | Any | 1 | Save resources |
| **Staging** | Any | 2 | Test HA behavior |
| **Prod** | Stateless API | 3 (min) | HA + rolling updates + zone spread |
| **Prod** | Frontend/BFF | 2-5 | Based on traffic |
| **Prod** | Auth/Payment (critical) | 3-5 | Zero downtime, PDB |
| **Prod** | Background workers | 2-3 | Can tolerate brief downtime |
| **Prod** | Database (StatefulSet) | 3 | Primary + 2 replicas for quorum |
| **Prod** | Redis Cache | 3 (sentinel) or 6 (cluster) | Depends on mode |

### PodDisruptionBudget — Protect Your Replicas

> **"Never let Kubernetes take down too many pods at once"**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2          # Always keep at least 2 pods running
  # OR
  # maxUnavailable: 1      # At most 1 pod can be down
  selector:
    matchLabels:
      app: my-api
```

**When PDB kicks in:**
- Node drain (`kubectl drain node1`) — PDB prevents draining if it would violate the budget
- Cluster autoscaler scaling down nodes
- Voluntary disruptions (upgrades, maintenance)

**PDB does NOT protect against:** Pod crashes, OOMKills, node failures (involuntary disruptions)

---

### Anti-Affinity: Spread Pods Across Nodes

> **"Don't put all your eggs in one basket (node)"**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-api
              topologyKey: "kubernetes.io/hostname"   # Different nodes
      # For cross-zone spread:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-api
```

---

## 📌 Part 3: CPU & Memory — How to Decide Requests and Limits

---

### The Golden Rule

```
┌──────────────────────────────────────────────────────────────────────┐
│                   RESOURCE DECISION FRAMEWORK                        │
│                                                                       │
│   Step 1: Deploy with NO limits, observe for 3-7 days               │
│   Step 2: Check actual usage with metrics-server/Prometheus          │
│   Step 3: Set REQUESTS = P95 of actual usage                        │
│   Step 4: Set LIMITS = 2× to 3× of requests (breathing room)       │
│                                                                       │
│   ┌────────────────────────────────────────────────────────────┐    │
│   │  Example: App uses ~200m CPU and ~256Mi memory normally     │    │
│   │                                                             │    │
│   │  requests:                                                  │    │
│   │    cpu: "200m"       ← Guaranteed (scheduler uses this)     │    │
│   │    memory: "256Mi"   ← Guaranteed minimum                   │    │
│   │                                                             │    │
│   │  limits:                                                    │    │
│   │    cpu: "500m"       ← Can burst up to this                 │    │
│   │    memory: "512Mi"   ← OOMKilled if exceeds this           │    │
│   └────────────────────────────────────────────────────────────┘    │
│                                                                       │
│   NEVER set limits = requests in production unless you want          │
│   "Guaranteed" QoS (databases, critical infra only)                  │
└──────────────────────────────────────────────────────────────────────┘
```

### Commands to Check Actual Resource Usage

```bash
# Current pod resource usage
kubectl top pods -n production

# Node resource usage
kubectl top nodes

# Detailed pod resource usage
kubectl describe pod <pod-name> | grep -A 5 "Requests\|Limits"

# Get resource usage over time (requires Prometheus)
# Query: container_memory_usage_bytes{namespace="production"}
# Query: rate(container_cpu_usage_seconds_total{namespace="production"}[5m])
```

### QoS Classes — What Kubernetes Assigns Automatically

```
┌──────────────────────────────────────────────────────────────────────┐
│              QUALITY OF SERVICE (QoS) CLASSES                        │
│                                                                       │
│  Guaranteed (Best protection — killed LAST)                          │
│  ├── requests == limits for BOTH cpu and memory                      │
│  └── Use for: Databases, payment services, critical infra            │
│                                                                       │
│  Burstable (Middle priority)                                         │
│  ├── requests < limits (at least one resource has requests set)      │
│  └── Use for: Most production workloads (APIs, web servers)          │
│                                                                       │
│  BestEffort (First to be killed — no protection)                     │
│  ├── No requests or limits set at all                                │
│  └── Use for: Batch jobs, dev environments, non-critical tasks       │
│                                                                       │
│  When node runs low on memory, K8s kills in order:                   │
│  BestEffort → Burstable → Guaranteed                                 │
└──────────────────────────────────────────────────────────────────────┘
```

### Real-World Resource Sizing Examples

| Application Type | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------------|-------------|-----------|----------------|--------------|
| **Nginx/Static** | 50m | 200m | 64Mi | 128Mi |
| **Node.js API** | 100m-250m | 500m-1000m | 128Mi-256Mi | 512Mi |
| **Java/Spring Boot** | 250m-500m | 1000m-2000m | 512Mi-1Gi | 1Gi-2Gi |
| **Python/Django** | 100m-250m | 500m-1000m | 128Mi-256Mi | 512Mi |
| **Go API** | 50m-100m | 250m-500m | 32Mi-64Mi | 128Mi-256Mi |
| **PostgreSQL** | 250m-500m | 1000m-2000m | 256Mi-512Mi | 1Gi-4Gi |
| **Redis** | 100m | 500m | 128Mi-256Mi | 512Mi-1Gi |
| **Elasticsearch** | 500m-1000m | 2000m | 2Gi | 4Gi-8Gi |

> **⚠️ Important:** These are STARTING POINTS. Always measure actual usage and adjust.

### Common Mistake: Over-Provisioning

```
┌──────────────────────────────────────────────────────────────────────┐
│              THE OVER-PROVISIONING TRAP                               │
│                                                                       │
│  Team sets: requests.cpu=1000m, requests.memory=2Gi                  │
│  Actual usage: cpu=50m, memory=200Mi                                 │
│                                                                       │
│  Result:                                                              │
│  • Node with 4 CPU can only fit 4 pods (scheduler reserves 1 CPU    │
│    per pod even though each uses only 50m)                           │
│  • You're paying for 20× more resources than needed                  │
│  • Cluster autoscaler spins up unnecessary nodes                     │
│                                                                       │
│  Fix: Use VPA in "Off" mode to get recommendations, then apply      │
│  kubectl describe vpa <name> → see recommended values               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 📌 Part 4: HPA vs VPA vs Manual Scaling — When to Use What

---

### Decision Tree

```
┌──────────────────────────────────────────────────────────────────────┐
│                SCALING DECISION TREE                                  │
│                                                                       │
│  Is your traffic predictable?                                        │
│  ├── YES, constant → Manual scaling (fixed replicas)                 │
│  ├── YES, time-based (9am spike) → CronJob + manual (or KEDA)       │
│  └── NO, unpredictable                                               │
│       ├── Is your app stateless? → HPA (add more pods)              │
│       └── Is your app stateful/can't scale horizontally?             │
│            └── VPA (make pods bigger)                                │
│                                                                       │
│  Can you use both HPA + VPA?                                        │
│  └── YES, BUT: Never use both on CPU/memory simultaneously.         │
│      Use HPA on CPU, VPA on memory, or use Multidimensional VPA.    │
└──────────────────────────────────────────────────────────────────────┘
```

### HPA — Horizontal Pod Autoscaler (More Pods)

**Best for:** Stateless apps, APIs, web servers, workers

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Wait 60s before scaling up more
      policies:
      - type: Pods
        value: 4                        # Add max 4 pods at a time
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 25                       # Remove max 25% pods at a time
        periodSeconds: 60
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70          # Scale when avg CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80          # Also watch memory
```

**HPA Pro Tips:**
1. **Set `scaleDown` stabilization to 300s+** — prevents flapping (up-down-up-down)
2. **Target 60-80% CPU**, not 50% — leaves room but doesn't over-scale
3. **Use `behavior` policies** — control HOW FAST scaling happens
4. **HPA needs metrics-server** — `kubectl top pods` must work first

### VPA — Vertical Pod Autoscaler (Bigger Pods)

**Best for:** Stateful apps, databases, JVM apps with varying heap, single-replica services

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: db-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: postgres
  updatePolicy:
    updateMode: "Off"     # Start with "Off" to just get recommendations!
  resourcePolicy:
    containerPolicies:
    - containerName: postgres
      minAllowed:
        cpu: 100m
        memory: 256Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
```

**VPA Modes:**

| Mode | What it does | Use when |
|------|-------------|----------|
| **Off** | Only gives recommendations | First time setup, learning actual usage |
| **Initial** | Sets resources on pod creation only | Don't want restarts |
| **Auto** | Restarts pods to apply new resources | Databases, stateful apps |

**⚠️ VPA Warning:** Auto mode RESTARTS pods. For databases, use "Off" mode and apply manually.

### Manual Scaling — When it's the RIGHT choice

```yaml
# Fixed scaling for consistent workloads
apiVersion: apps/v1
kind: Deployment
metadata:
  name: internal-tool
spec:
  replicas: 2     # Internal tool, used by 20 employees, traffic is constant
```

**Use manual when:**
- Internal tools with predictable user count
- Background processors with fixed queue rate
- Services behind a rate limiter
- Cost-sensitive environments where you want full control

### KEDA — Event-Driven Autoscaling (Advanced)

**Best for:** Queue-based workers, event-driven architectures

```yaml
# Scale based on RabbitMQ queue length
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 1
  maxReplicaCount: 30
  triggers:
  - type: rabbitmq
    metadata:
      queueName: orders
      queueLength: "5"        # 1 pod per 5 messages in queue
```

---

## 📌 Part 5: Deployment vs StatefulSet vs DaemonSet — Choose Correctly

---

### The Decision Matrix

```
┌──────────────────────────────────────────────────────────────────────┐
│            WORKLOAD TYPE DECISION                                     │
│                                                                       │
│  Does your app need:                                                 │
│                                                                       │
│  Stable network identity? (same hostname always)                     │
│  ├── NO  → Deployment                                               │
│  └── YES                                                             │
│       Ordered startup/shutdown?                                      │
│       ├── NO  → Deployment (probably)                               │
│       └── YES                                                        │
│            Persistent storage per pod?                                │
│            ├── NO  → Deployment with PVC                            │
│            └── YES → StatefulSet ✓                                  │
│                                                                       │
│  Must run on EVERY node?                                             │
│  └── YES → DaemonSet ✓                                              │
│                                                                       │
│  Short-lived task (run once and exit)?                                │
│  └── YES → Job / CronJob ✓                                          │
└──────────────────────────────────────────────────────────────────────┘
```

### Comparison Table

| Feature | Deployment | StatefulSet | DaemonSet |
|---------|-----------|-------------|-----------|
| **Pod Names** | Random (`app-7f8b4-xk2lm`) | Ordered (`db-0`, `db-1`, `db-2`) | One per node |
| **Scaling** | Any order | Sequential (0→1→2) | Auto (node count) |
| **Storage** | Shared PVC | Per-pod PVC (volumeClaimTemplates) | Usually hostPath |
| **Network** | Random IP | Stable DNS (`db-0.db-svc`) | Node IP |
| **Rolling Update** | Any order | Reverse order (2→1→0) | One per node |
| **Use Cases** | APIs, web apps, workers | Databases, Kafka, ZooKeeper | Logging, monitoring, networking |

### When to use StatefulSet (Real Examples)

```yaml
# PostgreSQL with StatefulSet — each pod gets its own 10Gi volume
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres         # Required! Creates DNS entries
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:          # Each pod gets its OWN PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 10Gi
```

**DNS created:**
- `postgres-0.postgres.default.svc.cluster.local`
- `postgres-1.postgres.default.svc.cluster.local`
- `postgres-2.postgres.default.svc.cluster.local`

### When to use DaemonSet (Real Examples)

```yaml
# Fluentd log collector — must run on EVERY node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule       # Even run on master nodes
      containers:
      - name: fluentd
        image: fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

**Use DaemonSet for:**
- Log collectors (Fluentd, Filebeat)
- Node monitoring (Prometheus Node Exporter, Datadog agent)
- Network plugins (Calico, Cilium)
- Storage plugins (CSI drivers)

---

## 📌 Part 6: Designing Resources Based on Traffic, Cost & Performance

---

### The Production Resource Planning Workflow

```
┌──────────────────────────────────────────────────────────────────────┐
│              RESOURCE PLANNING (Step-by-Step)                         │
│                                                                       │
│  Step 1: ESTIMATE                                                    │
│  ├── Expected RPS (requests per second)?                             │
│  ├── Each pod can handle how many RPS? (load test to find out)       │
│  └── Pods needed = Expected RPS / RPS per pod                        │
│                                                                       │
│  Step 2: CAPACITY PLAN                                               │
│  ├── Add 30% buffer for traffic spikes                               │
│  ├── Add 1-2 pods for rolling update headroom                        │
│  └── Consider zone distribution (3 zones = replicas divisible by 3)  │
│                                                                       │
│  Step 3: NODE SIZING                                                 │
│  ├── Small pods (< 500m CPU) → Use larger, fewer nodes              │
│  │   (less overhead, better bin-packing)                             │
│  ├── Large pods (> 2 CPU) → Use dedicated node pools                │
│  └── Leave 10-15% node capacity for system pods (kube-system)       │
│                                                                       │
│  Step 4: COST OPTIMIZE                                               │
│  ├── Use Spot/Preemptible instances for stateless workloads          │
│  ├── Use Reserved Instances for baseline (always-on) pods            │
│  ├── Cluster Autoscaler for node scaling                             │
│  └── HPA for pod scaling                                             │
└──────────────────────────────────────────────────────────────────────┘
```

### Real-World Scenario: E-Commerce Platform

```
┌──────────────────────────────────────────────────────────────────────┐
│           SCENARIO: E-Commerce (10,000 daily users, 500 peak RPS)   │
│                                                                       │
│  Service          │ Replicas │ CPU Req/Lim │ Mem Req/Lim │ Why       │
│  ─────────────────┼──────────┼─────────────┼─────────────┼────────── │
│  Frontend (React) │ 3        │ 100m/300m   │ 128Mi/256Mi │ Static    │
│  API Gateway      │ 3        │ 250m/1000m  │ 256Mi/512Mi │ Auth+rout │
│  Order Service    │ 3-10(HPA)│ 200m/500m   │ 256Mi/512Mi │ Bursty   │
│  Payment Service  │ 3        │ 200m/500m   │ 256Mi/512Mi │ Critical  │
│  User Service     │ 2        │ 100m/300m   │ 128Mi/256Mi │ Low traffic│
│  Notification Svc │ 2        │ 100m/300m   │ 128Mi/256Mi │ Async     │
│  PostgreSQL       │ 3 (SS)   │ 500m/2000m  │ 1Gi/4Gi    │ StatefulS │
│  Redis            │ 3 (SS)   │ 100m/500m   │ 256Mi/1Gi  │ Cache     │
│  RabbitMQ         │ 3 (SS)   │ 250m/1000m  │ 512Mi/1Gi  │ Queue     │
│                                                                       │
│  Node Pool Strategy:                                                  │
│  • System pool: 2 × t3.medium (on-demand) — for kube-system         │
│  • App pool: 3-6 × t3.xlarge (spot) — for stateless services        │
│  • Data pool: 3 × r5.large (on-demand) — for databases              │
│                                                                       │
│  Estimated monthly cost (AWS EKS):                                   │
│  • EKS control plane: $73                                            │
│  • EC2 instances: ~$300-500 (with spot savings)                      │
│  • EBS volumes: ~$50                                                 │
│  • Load Balancer: ~$20                                               │
│  • TOTAL: ~$450-650/month                                            │
└──────────────────────────────────────────────────────────────────────┘
```

### Spot Instances Strategy

```yaml
# Node pool with spot instances (EKS)
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: prod-cluster
  region: ap-south-1
managedNodeGroups:
  - name: spot-apps
    instanceTypes: ["t3.large", "t3.xlarge", "m5.large"]  # Multiple types!
    spot: true
    minSize: 2
    maxSize: 10
    labels:
      lifecycle: spot
    taints:
    - key: spot
      value: "true"
      effect: NoSchedule
  - name: ondemand-data
    instanceTypes: ["r5.large"]
    spot: false
    minSize: 3
    maxSize: 3
    labels:
      lifecycle: on-demand
```

**Spot Rules:**
- ✅ Stateless apps (APIs, web servers) → Spot instances (65-90% savings!)
- ❌ Databases, StatefulSets → NEVER on Spot
- ✅ Use multiple instance types → AWS has more capacity to give you
- ✅ Use Pod Disruption Budgets → graceful handling when Spot is reclaimed

---

## 📌 Part 7: Common Production Mistakes & Best Practices

---

### ❌ Mistake 1: No Resource Requests/Limits

```
Problem: Pod scheduled on overloaded node → OOMKilled → cascading failures
Fix:     ALWAYS set at least resource requests

# Bad
containers:
- name: app
  image: myapp

# Good
containers:
- name: app
  image: myapp
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      memory: 256Mi      # CPU limits are debatable — see below
```

**The CPU Limits Debate:**
- Google's internal recommendation: **Don't set CPU limits**, only set CPU requests
- Reason: CPU is a compressible resource (throttled, not killed). Limits cause unnecessary throttling
- **Memory limits: ALWAYS set them.** Memory is incompressible (OOMKilled)

---

### ❌ Mistake 2: No Health Checks (Probes)

```
Problem: App deadlocked, still showing "Running" status
         K8s sends traffic to a pod that can't respond
         Users see 5xx errors

Fix: Configure all 3 probes

# Minimum viable probes for any production app
startupProbe:             # Let the app start
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30    # 30 × 10s = 5 minutes to start
  periodSeconds: 10

livenessProbe:            # Restart if dead
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:           # Remove from traffic if not ready
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 3
```

---

### ❌ Mistake 3: Using `latest` Tag in Production

```
Problem: "It worked yesterday, same image!"
         → But :latest now points to a different build
         → Different pods running different versions

Fix: Always use specific tags

# Bad
image: myapp:latest

# Good
image: myapp:v2.3.1-abc1234    # version + git sha

# Also set:
imagePullPolicy: IfNotPresent   # Don't re-pull if already exists
```

---

### ❌ Mistake 4: No Pod Disruption Budgets

```
Problem: kubectl drain node1 → ALL pods on node1 killed simultaneously
         → 100% of your API pods were on that node → total downtime

Fix: PDB + Anti-Affinity (already explained above)
```

---

### ❌ Mistake 5: Secrets in Plain ConfigMaps or Environment Variables

```
Problem: Database password in plain text ConfigMap
         → Anyone with namespace access can read it
         → Shows up in `kubectl describe pod`

Fix: Use Kubernetes Secrets (minimum) or external secret managers

# Better: External Secrets Operator with AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-secret
  data:
  - secretKey: password
    remoteRef:
      key: /production/database/password
```

---

### ❌ Mistake 6: No Namespace Separation

```
Problem: Dev and prod workloads in default namespace
         → Dev team accidentally deletes prod pods
         → No resource isolation

Fix: Namespace strategy

# Production namespace structure
├── production          # Prod workloads
│   ├── ResourceQuota
│   └── NetworkPolicy (restrict ingress)
├── staging             # Pre-prod testing
├── development         # Dev workloads (lower quotas)
├── monitoring          # Prometheus, Grafana
├── ingress-system      # Ingress controllers
└── cert-manager        # TLS certificate management
```

---

### ❌ Mistake 7: No Graceful Shutdown

```
Problem: Pod killed during rolling update → in-flight requests dropped
         → Users see 502/503 errors

Fix: Handle SIGTERM + use preStop hook

containers:
- name: app
  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 10"]    # Wait for LB to drain
  terminationGracePeriodSeconds: 30               # Total time to shut down
```

**What happens during graceful shutdown:**
1. K8s sends SIGTERM to container
2. K8s removes pod from Service endpoints (no new traffic)
3. `preStop` hook runs (sleep 10 lets in-flight requests finish)
4. App receives SIGTERM and shuts down
5. After 30s (`terminationGracePeriodSeconds`), K8s sends SIGKILL

---

### ❌ Mistake 8: No Network Policies (Everything Can Talk to Everything)

```yaml
# Restrict: Only frontend can talk to API, only API can talk to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-server        # Only api-server pods can reach DB
    ports:
    - protocol: TCP
      port: 5432
```

---

## 📌 Part 8: Production Readiness Checklist

```
┌──────────────────────────────────────────────────────────────────────┐
│           PRODUCTION READINESS CHECKLIST                              │
│                                                                       │
│  Resource Management                                                 │
│  ☐ Resource requests set for all containers                          │
│  ☐ Memory limits set for all containers                              │
│  ☐ ResourceQuotas per namespace                                      │
│  ☐ LimitRanges for default resource policies                        │
│                                                                       │
│  High Availability                                                    │
│  ☐ Minimum 3 replicas for critical services                          │
│  ☐ PodDisruptionBudgets configured                                   │
│  ☐ Pod anti-affinity for cross-node spread                           │
│  ☐ TopologySpreadConstraints for cross-zone spread                   │
│                                                                       │
│  Health & Reliability                                                 │
│  ☐ Liveness, Readiness, and Startup probes configured                │
│  ☐ Graceful shutdown (preStop + terminationGracePeriod)              │
│  ☐ Rolling update strategy with maxSurge/maxUnavailable              │
│                                                                       │
│  Security                                                             │
│  ☐ No containers running as root                                     │
│  ☐ Read-only root filesystem where possible                          │
│  ☐ NetworkPolicies restricting pod-to-pod traffic                    │
│  ☐ Secrets in external secret manager (not plain ConfigMaps)         │
│  ☐ RBAC with least-privilege access                                   │
│  ☐ Image scanning in CI/CD pipeline                                  │
│                                                                       │
│  Scaling                                                              │
│  ☐ HPA for variable-traffic stateless services                       │
│  ☐ Cluster Autoscaler configured                                     │
│  ☐ Spot instances for stateless workloads (cost savings)             │
│                                                                       │
│  Observability                                                        │
│  ☐ Prometheus + Grafana for metrics                                  │
│  ☐ Centralized logging (EFK/Loki)                                    │
│  ☐ Distributed tracing (Jaeger/Tempo)                                │
│  ☐ Alerting rules for SLA violations                                 │
│                                                                       │
│  CI/CD                                                                │
│  ☐ GitOps (ArgoCD/FluxCD) for deployments                           │
│  ☐ Specific image tags (never :latest)                               │
│  ☐ Canary or blue-green deployment strategy                          │
│  ☐ Automated rollback on failure                                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 📌 Part 9: Quick-Fire Interview Q&A (Rapid Round)

| # | Question | Short Answer |
|---|----------|-------------|
| 1 | Difference between Deployment and ReplicaSet? | Deployment manages ReplicaSets. ReplicaSet manages Pods. Never create RS directly. |
| 2 | What is init container? | Runs BEFORE main container. Use for: DB migration, config fetch, dependency check. |
| 3 | What is sidecar pattern? | Helper container alongside main container. Ex: logging agent, proxy (Envoy). |
| 4 | How to do zero-downtime deployment? | Rolling update (default) + readiness probes + PDB + preStop hook. |
| 5 | What is headless service? | `clusterIP: None`. No load balancing. Returns all pod IPs. For StatefulSet DNS. |
| 6 | ConfigMap vs Secret? | Same concept. Secrets are base64 encoded (not encrypted!). Use external KMS for real security. |
| 7 | What happens if ETCD dies? | Entire cluster is brain-dead. No new operations. Existing pods keep running but can't be managed. |
| 8 | Difference between kubectl exec and kubectl attach? | exec = start new process in container. attach = connect to existing PID 1. |
| 9 | What is a Finalizer? | Pre-delete hook. Prevents resource deletion until cleanup is done (e.g., delete external LB). |
| 10 | How does DNS work in K8s? | CoreDNS creates entries: `<svc>.<ns>.svc.cluster.local`. Pods resolve via `/etc/resolv.conf`. |
| 11 | What is `kubectl port-forward`? | Tunnels local port to pod port. For debugging only, NOT for production traffic. |
| 12 | Static Pods vs Regular Pods? | Static Pods managed by Kubelet directly (from `/etc/kubernetes/manifests`). Used for control plane components. |
| 13 | What is a CRD? | Custom Resource Definition. Extends K8s API. Used by operators (Prometheus, ArgoCD). |
| 14 | How to debug a CrashLoopBackOff? | `kubectl logs <pod> --previous` → see crash reason. Check probes, resources, image, config. |
| 15 | What is Helm? | Package manager for K8s. Charts = reusable templates. `helm install` instead of 50 kubectl commands. |

---

## 🧠 Memory Shortcuts for This Chapter

### Resource Decisions: **"ROML"**
```
R = Requests (guaranteed minimum — set to P95 actual usage)
O = Observe first (deploy without limits, measure for 7 days)
M = Memory limits always (incompressible — OOMKilled if exceeded)
L = Leave CPU limits off (compressible — Google's recommendation)
```

### Replica Count: **"3-2-1"**
```
3 = Production critical services (minimum 3 replicas)
2 = Staging and less-critical services
1 = Development only
```

### Scaling Strategy: **"HSM"**
```
H = HPA for unpredictable, stateless workloads
S = Static (manual) for predictable, constant workloads
M = Mix HPA + manual (set minReplicas = baseline, let HPA handle spikes)
```

### Production Checklist: **"PRSH-O"**
```
P = Probes (liveness + readiness + startup)
R = Resources (requests + limits)
S = Security (RBAC + NetworkPolicy + Secrets)
H = HA (replicas + PDB + anti-affinity)
O = Observability (metrics + logs + traces)
```

---

**← Previous: [15 - Practice Workbook](./15-practice-workbook.md)** | **[Back to Index](./00-kubernetes-index.md)** 🏠
