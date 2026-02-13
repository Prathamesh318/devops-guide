# 📊 Chapter 12: Monitoring & Logging

> **"You can't fix what you can't see!"**

---

## 🎯 Observability Overview

```
┌────────────────────────────────────────────────────────────────┐
│                 THREE PILLARS OF OBSERVABILITY                  │
│                                                                 │
│   ┌────────────┐   ┌────────────┐   ┌────────────┐            │
│   │   LOGS     │   │  METRICS   │   │   TRACES   │            │
│   │            │   │            │   │            │            │
│   │  What      │   │  How much  │   │  Where     │            │
│   │  happened  │   │  (numbers) │   │  (flow)    │            │
│   └────────────┘   └────────────┘   └────────────┘            │
│                                                                 │
│   Tools:                                                       │
│   • Logs: Fluentd, Loki, ELK                                  │
│   • Metrics: Prometheus, Grafana                              │
│   • Traces: Jaeger, Zipkin                                    │
└────────────────────────────────────────────────────────────────┘
```

---

## 🖥️ Kubernetes Dashboard

> **Built-in web UI for cluster management**

### Enable Dashboard

```bash
# Minikube
minikube dashboard

# KIND / Kubeadm
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Access
kubectl proxy
# Open: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

### Create Dashboard Admin User

```yaml
# dashboard-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

```bash
kubectl apply -f dashboard-admin.yaml

# Get token
kubectl -n kubernetes-dashboard create token admin-user
```

---

## 📈 Metrics Server

> **Collects resource metrics (CPU, memory) from nodes and pods**

```bash
# Install metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For local clusters (add --kubelet-insecure-tls)
# Edit deployment and add flag to container args

# View metrics
kubectl top nodes
kubectl top pods
kubectl top pods --containers
```

---

## 🔥 Prometheus & Grafana

> **Industry-standard monitoring stack**

```
┌────────────────────────────────────────────────────────────────┐
│                PROMETHEUS ARCHITECTURE                          │
│                                                                 │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐              │
│   │  Targets │────►│Prometheus│────►│ Grafana  │              │
│   │  (pods)  │scrape│  (Store) │query│  (View)  │              │
│   └──────────┘     └────┬─────┘     └──────────┘              │
│                         │                                       │
│                         ▼                                       │
│                  ┌─────────────┐                               │
│                  │AlertManager │                               │
│                  │  (Alerts)   │                               │
│                  └─────────────┘                               │
└────────────────────────────────────────────────────────────────┘
```

### Install with Helm

```bash
# Add Prometheus repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus stack (includes Grafana)
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# Access Grafana
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
# Default: admin / prom-operator
```

---

## 📝 Logging with kubectl

```bash
# Pod logs
kubectl logs pod-name
kubectl logs pod-name -c container-name  # specific container
kubectl logs pod-name --previous         # previous instance
kubectl logs pod-name -f                  # follow/stream
kubectl logs pod-name --tail=100          # last 100 lines
kubectl logs pod-name --since=1h          # last hour

# All pods with label
kubectl logs -l app=nginx

# Deployment logs
kubectl logs deployment/my-app
```

---

## 📚 Centralized Logging (EFK/Loki)

### EFK Stack (Elasticsearch, Fluentd, Kibana)

```
┌────────────────────────────────────────────────────────────────┐
│                    EFK STACK                                    │
│                                                                 │
│   Pod Logs ──► Fluentd ──► Elasticsearch ──► Kibana           │
│                (collect)      (store)        (view)            │
│                                                                 │
│   Fluentd runs as DaemonSet on each node                      │
│   Collects logs from /var/log/containers/                     │
└────────────────────────────────────────────────────────────────┘
```

### Grafana Loki (Lightweight Alternative)

```bash
# Install Loki stack
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack -n logging --create-namespace
```

---

## 🔍 Debugging Commands

```bash
# Cluster info
kubectl cluster-info
kubectl get componentstatuses

# Node debugging
kubectl describe node node-name
kubectl get events --sort-by='.lastTimestamp'

# Pod debugging
kubectl describe pod pod-name
kubectl get pod pod-name -o yaml
kubectl exec -it pod-name -- /bin/sh

# Resource usage
kubectl top nodes
kubectl top pods

# API server logs (control plane)
kubectl logs -n kube-system kube-apiserver-master
```

---

## 🚨 Common Issues & Solutions

| Issue | Debugging Commands |
|-------|-------------------|
| Pod not starting | `kubectl describe pod`, `kubectl get events` |
| CrashLoopBackOff | `kubectl logs pod --previous` |
| ImagePullBackOff | Check image name, registry credentials |
| Pending pod | `kubectl describe pod` (check events, resources) |
| Service not working | `kubectl get endpoints`, check selectors |

---

## 🧠 Memory Shortcuts

### Observability Pillars: **"LMT"**
```
L = Logs (what happened)
M = Metrics (how much, numbers)
T = Traces (request flow)
```

### Debugging Order: **"DELP"**
```
D = Describe (kubectl describe)
E = Events (kubectl get events)
L = Logs (kubectl logs)
P = Pod shell (kubectl exec)
```

---

**Next: [13 - Hands-on Projects](./13-hands-on-projects.md)** ➡️
