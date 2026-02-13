# 📋 Chapter 14: Kubernetes Cheat Sheet

> **"Your ultimate quick reference guide!"**

---

## ⚡ Essential Aliases (Add to ~/.bashrc)

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kga='kubectl get all'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias ke='kubectl exec -it'

# Namespace shortcuts
alias kn='kubectl config set-context --current --namespace'
alias kgpa='kubectl get pods --all-namespaces'
```

---

## 📦 Core Commands

### Cluster Info
```bash
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl top nodes
```

### Namespace Operations
```bash
kubectl get ns
kubectl create ns <name>
kubectl delete ns <name>
kubectl config set-context --current --namespace=<name>
```

### Pod Operations
```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods -w                  # Watch mode
kubectl get pods --show-labels
kubectl get pods -l app=nginx        # By label
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> -f                # Follow
kubectl logs <pod> -c <container>    # Specific container
kubectl logs <pod> --previous        # Previous instance
kubectl exec -it <pod> -- /bin/bash
kubectl port-forward <pod> 8080:80
kubectl delete pod <pod>
kubectl run nginx --image=nginx      # Create pod
```

### Deployment Operations
```bash
kubectl get deployments
kubectl get deploy                   # Short
kubectl describe deploy <name>
kubectl create deploy nginx --image=nginx --replicas=3
kubectl scale deploy <name> --replicas=5
kubectl set image deploy/<name> <container>=<image>
kubectl rollout status deploy/<name>
kubectl rollout history deploy/<name>
kubectl rollout undo deploy/<name>
kubectl rollout undo deploy/<name> --to-revision=2
kubectl rollout restart deploy/<name>
kubectl delete deploy <name>
```

### Service Operations
```bash
kubectl get svc
kubectl describe svc <name>
kubectl expose deploy <name> --port=80 --type=NodePort
kubectl delete svc <name>
```

---

## 📋 Resource Types Quick Reference

| Short | Full Name | Description |
|-------|-----------|-------------|
| `po` | pods | Running containers |
| `svc` | services | Network endpoints |
| `deploy` | deployments | Manages ReplicaSets |
| `rs` | replicasets | Maintains pod count |
| `ds` | daemonsets | One pod per node |
| `sts` | statefulsets | Stateful apps |
| `cm` | configmaps | Configuration data |
| `secret` | secrets | Sensitive data |
| `pv` | persistentvolumes | Storage |
| `pvc` | persistentvolumeclaims | Storage requests |
| `sc` | storageclasses | Storage types |
| `ns` | namespaces | Virtual clusters |
| `no` | nodes | Cluster machines |
| `ing` | ingresses | HTTP routing |
| `hpa` | horizontalpodautoscalers | Auto-scaling |
| `sa` | serviceaccounts | Pod identities |
| `crd` | customresourcedefinitions | Custom resources |

---

## 🔍 Filtering & Output

```bash
# Output formats
kubectl get pods -o wide              # More columns
kubectl get pods -o yaml              # YAML format
kubectl get pods -o json              # JSON format
kubectl get pods -o name              # Names only

# Label selectors
kubectl get pods -l app=nginx
kubectl get pods -l 'env in (prod,staging)'
kubectl get pods -l app=nginx,tier=frontend

# Field selectors
kubectl get pods --field-selector=status.phase=Running

# Sort
kubectl get pods --sort-by='.metadata.creationTimestamp'

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase
```

---

## 🧠 Memory Shortcuts Summary

### Architecture: **ACES + KKC**
```
Control Plane:              Worker Node:
A = API Server              K = Kubelet
C = Controller Manager      K = Kube-proxy
E = ETCD                    C = Container Runtime
S = Scheduler
```

### Workloads: **DJSCD**
```
D = Deployment
J = Job
S = StatefulSet
C = CronJob
D = DaemonSet
```

### Services: **CNLE**
```
C = ClusterIP (internal, default)
N = NodePort (external via node)
L = LoadBalancer (cloud LB)
E = ExternalName (DNS alias)
```

### Storage: **SCP**
```
S = StorageClass (provisioning rules)
C = Claim (PVC - what pod wants)
P = PersistentVolume (actual storage)
```

### Probes: **LSR**
```
L = Liveness (restart if fails)
S = Startup (wait for it)
R = Readiness (remove from service)
```

### RBAC: **RRBB**
```
R = Role (namespace scope)
R = ClusterRole (cluster scope)
B = RoleBinding
B = ClusterRoleBinding
```

### Debugging: **DELP**
```
D = Describe
E = Events
L = Logs
P = Pod shell (exec)
```

---

## 📊 YAML Templates

### Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: myapp
spec:
  containers:
  - name: container
    image: nginx
    ports:
    - containerPort: 80
```

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
```

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
```

### ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  KEY: value
```

### Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  password: secret123
```

---

## 🚨 Common Troubleshooting

| Problem | Commands |
|---------|----------|
| Pod not starting | `kubectl describe pod <name>` |
| CrashLoopBackOff | `kubectl logs <pod> --previous` |
| ImagePullBackOff | Check image name, `kubectl get events` |
| Pending | Check resources, `kubectl describe pod` |
| Service not working | `kubectl get endpoints <svc>` |

---

## 🎯 Exam/Interview Quick Prep

```
1. What is Kubernetes?
   → Container orchestration platform for automating deployment, scaling, and management

2. Control Plane components?
   → API Server, ETCD, Scheduler, Controller Manager

3. Pod vs Container?
   → Pod is smallest deployable unit, can contain multiple containers with shared network/storage

4. Deployment vs StatefulSet?
   → Deployment: stateless, random pod names
   → StatefulSet: stateful, ordered names, stable network identity

5. Service types?
   → ClusterIP, NodePort, LoadBalancer, ExternalName

6. How does K8s self-heal?
   → Controller Manager watches state, recreates pods if they die

7. ConfigMap vs Secret?
   → ConfigMap: non-sensitive config
   → Secret: sensitive data (base64 encoded)

8. What is RBAC?
   → Role-Based Access Control - defines who can do what on which resources
```

---

## 🔗 Useful Resources

- **Official Docs:** https://kubernetes.io/docs/
- **kubectl Reference:** https://kubernetes.io/docs/reference/kubectl/
- **Interactive Tutorial:** https://kubernetes.io/docs/tutorials/
- **K8s the Hard Way:** https://github.com/kelseyhightower/kubernetes-the-hard-way

---

**🎉 Congratulations! You've completed the Kubernetes learning guide!**

**Now practice everything: [15 - Practice Workbook (20 Exercises)](./15-practice-workbook.md)** 🏋️

**← Back to [Index](./00-kubernetes-index.md)**
