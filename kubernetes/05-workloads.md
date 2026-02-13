# ⚙️ Chapter 5: Workload Resources

> **"Workloads define HOW your containers run and scale"**

---

## 🎯 Workload Hierarchy

```
                    Deployment
                        │
                        │ manages
                        ▼
                    ReplicaSet
                        │
                        │ creates
                        ▼
                    Pod(s)
                        │
                        │ runs
                        ▼
                    Container(s)
```

**Memory: "DRS" (Drive!)** - Deployment → ReplicaSet → (Pod)S

---

## 🚀 Deployments

> **The standard way to run stateless applications**

### What Deployments Do
- ✅ Create and manage ReplicaSets
- ✅ Rolling updates (zero downtime)
- ✅ Rollback to previous versions
- ✅ Scale up/down
- ✅ Pause and resume deployments

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

### Deployment Commands

```bash
# Create deployment
kubectl create deployment nginx --image=nginx --replicas=3

# List deployments
kubectl get deployments
kubectl get deploy  # short

# Describe deployment
kubectl describe deployment nginx

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Update image (triggers rolling update)
kubectl set image deployment/nginx nginx=nginx:1.22

# Check rollout status
kubectl rollout status deployment/nginx

# Rollout history
kubectl rollout history deployment/nginx

# Rollback to previous
kubectl rollout undo deployment/nginx

# Rollback to specific revision
kubectl rollout undo deployment/nginx --to-revision=2

# Pause/Resume rollout
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx

# Delete deployment
kubectl delete deployment nginx
```

---

## 🔄 Rolling Updates

```
┌───────────────────────────────────────────────────────────────┐
│                    ROLLING UPDATE PROCESS                      │
│                                                                │
│   Before:  [v1] [v1] [v1] [v1]                                │
│                                                                │
│   Step 1:  [v1] [v1] [v1] [v2]  ← New pod created            │
│   Step 2:  [v1] [v1] [v2] [v2]  ← Old pod terminated         │
│   Step 3:  [v1] [v2] [v2] [v2]                                │
│   Step 4:  [v2] [v2] [v2] [v2]  ← Complete!                  │
│                                                                │
│   Zero downtime! Always some pods running                     │
└───────────────────────────────────────────────────────────────┘
```

### Rolling Update Strategy

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # Max pods that can be unavailable
      maxSurge: 1          # Max extra pods during update
```

| Strategy | Description |
|----------|-------------|
| **RollingUpdate** | Gradual replacement (default) |
| **Recreate** | Kill all old, then create new |

---

## 📊 ReplicaSets

> **Ensures specified number of pod replicas are running**

**Note:** Usually managed by Deployment, rarely created directly.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

### ReplicaSet Commands

```bash
kubectl get replicasets
kubectl get rs  # short
kubectl describe rs nginx-replicaset
kubectl scale rs nginx-replicaset --replicas=5
```

---

## 👹 DaemonSets

> **Runs one pod on EVERY node (or selected nodes)**

### Use Cases
- 📊 Log collectors (Fluentd, Logstash)
- 📈 Monitoring agents (Prometheus Node Exporter)
- 🔀 Network proxies (kube-proxy)
- 💾 Storage daemons (Ceph)

```
┌──────────────────────────────────────────────────────────────┐
│                     DAEMONSET CONCEPT                         │
│                                                               │
│   Node 1          Node 2          Node 3          Node 4     │
│   ┌─────┐         ┌─────┐         ┌─────┐         ┌─────┐   │
│   │ Pod │         │ Pod │         │ Pod │         │ Pod │   │
│   │ (D) │         │ (D) │         │ (D) │         │ (D) │   │
│   └─────┘         └─────┘         └─────┘         └─────┘   │
│                                                               │
│   One pod per node automatically!                            │
│   New node added → DaemonSet creates pod on it              │
└──────────────────────────────────────────────────────────────┘
```

### DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-daemonset
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:latest
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
```

### DaemonSet Commands

```bash
kubectl get daemonsets
kubectl get ds  # short
kubectl describe ds fluentd-daemonset
```

---

## 📋 Jobs

> **Run task to completion (one-time execution)**

### Use Cases
- 🗄️ Database migrations
- 📧 Batch email sending
- 🔢 Data processing
- 🧹 Cleanup tasks

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-job
spec:
  completions: 5      # Total successful completions needed
  parallelism: 2      # Pods running in parallel
  backoffLimit: 4     # Retries before failing
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never  # Required for Jobs
```

### Job Commands

```bash
kubectl get jobs
kubectl describe job pi-job
kubectl logs job/pi-job
kubectl delete job pi-job
```

---

## ⏰ CronJobs

> **Scheduled Jobs (like Linux cron)**

### Cron Schedule Format

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sun=0)
│ │ │ │ │
* * * * *
```

### Common Schedules

| Schedule | Meaning |
|----------|---------|
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour |
| `0 0 * * *` | Daily at midnight |
| `0 0 * * 0` | Weekly on Sunday |
| `0 0 1 * *` | Monthly on 1st |

### CronJob YAML

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cronjob
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-script:latest
            command: ["/bin/sh", "-c", "backup.sh"]
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  concurrencyPolicy: Forbid  # Don't run if previous still running
```

### CronJob Commands

```bash
kubectl get cronjobs
kubectl get cj  # short
kubectl describe cj backup-cronjob

# Manually trigger a CronJob
kubectl create job --from=cronjob/backup-cronjob manual-backup
```

---

## 🗃️ StatefulSets

> **For stateful applications needing stable identity**

### StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| **Pod Names** | Random (nginx-abc123) | Ordered (mysql-0, mysql-1) |
| **Storage** | Shared or none | Unique PVC per pod |
| **Scaling** | Any order | Sequential (0→1→2) |
| **Use Case** | Stateless apps | Databases, clusters |

```
┌───────────────────────────────────────────────────────────────┐
│                    STATEFULSET CONCEPT                         │
│                                                                │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│   │   mysql-0    │  │   mysql-1    │  │   mysql-2    │       │
│   │   (Master)   │  │   (Replica)  │  │   (Replica)  │       │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│          │                 │                 │                │
│   ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐       │
│   │    PVC-0     │  │    PVC-1     │  │    PVC-2     │       │
│   │  (unique)    │  │  (unique)    │  │  (unique)    │       │
│   └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                                │
│   • Stable network identity: mysql-0.mysql-svc                │
│   • Ordered scaling and updates                               │
│   • Persistent storage survives pod restart                   │
└───────────────────────────────────────────────────────────────┘
```

### StatefulSet YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-svc  # Headless service for DNS
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

---

## 📊 Comparison Summary

| Workload | Use Case | Pods | Storage |
|----------|----------|------|---------|
| **Deployment** | Stateless apps | Any | Shared |
| **StatefulSet** | Databases | Ordered | Unique |
| **DaemonSet** | Node agents | One/node | Local |
| **Job** | One-time task | Runs once | Temp |
| **CronJob** | Scheduled task | Periodic | Temp |

---

## 🧠 Memory Shortcuts

### Workload Types: **"DJSCD"**
```
D = Deployment (stateless, scaling)
J = Job (one-time tasks)
S = StatefulSet (databases)  
C = CronJob (scheduled)
D = DaemonSet (every node)
```

---

**Next: [06 - Storage](./06-storage.md)** ➡️
