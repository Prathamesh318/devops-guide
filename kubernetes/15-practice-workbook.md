# 🏋️ Chapter 15: Kubernetes Practice Workbook

> **"The best way to learn Kubernetes is to break things and fix them."**
> This workbook gives you 20 progressive hands-on exercises — work through them on Minikube or Kind.

---

## 🛠️ Setup: Get Your Local Cluster Ready

### Option A: Minikube (Recommended)

```bash
# Install minikube (Windows - using winget)
winget install Kubernetes.minikube

# Start cluster with enough resources
minikube start --driver=docker --cpus=2 --memory=4096

# Enable useful addons
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard

# Verify everything is running
kubectl get nodes
kubectl cluster-info
```

### Option B: Kind (Kubernetes in Docker)

```bash
# Install Kind
winget install Kubernetes.kind

# Create cluster
kind create cluster --name practice

# Verify
kubectl cluster-info --context kind-practice
```

### Useful Aliases (set these up first!)

```bash
# Add to your shell profile (.bashrc / .zshrc / PowerShell $PROFILE)
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kga='kubectl get all'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ka='kubectl apply -f'
alias kdel='kubectl delete -f'
```

---

## 🟢 Phase 1: Core Basics (Exercises 1–5)

> Master the fundamentals — Pods, Deployments, Services, Namespaces

---

### ✏️ Exercise 1: Your First Pod

> **Goal:** Create, inspect, and delete a Pod manually

**Step 1: Create a Pod imperatively**
```bash
# Create a simple nginx pod
kubectl run my-first-pod --image=nginx:latest --port=80

# Check if it's running
kubectl get pods

# Expected output:
# NAME           READY   STATUS    RESTARTS   AGE
# my-first-pod   1/1     Running   0          30s
```

**Step 2: Inspect the Pod**
```bash
# Get detailed information
kubectl describe pod my-first-pod

# Things to observe:
# ✅ Which node is it running on?
# ✅ What's the Pod IP?
# ✅ What events happened during creation?

# Check logs
kubectl logs my-first-pod

# Execute a command inside the pod
kubectl exec -it my-first-pod -- /bin/bash

# Inside the pod, run:
curl localhost:80    # Should show nginx welcome page
hostname             # Shows the pod name
exit
```

**Step 3: Create the SAME Pod declaratively (YAML)**
```bash
# First delete the imperative pod
kubectl delete pod my-first-pod

# Generate YAML template (pro tip!)
kubectl run my-first-pod --image=nginx:latest --port=80 --dry-run=client -o yaml > exercise1-pod.yaml
```

```yaml
# exercise1-pod.yaml — Create this file yourself!
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  labels:
    app: nginx
    exercise: "1"
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
# Apply and verify
kubectl apply -f exercise1-pod.yaml
kubectl get pods -o wide
kubectl get pod my-first-pod -o yaml   # See full YAML with defaults filled in
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise1-pod.yaml
```

> **✅ Checkpoint:** Can you explain the difference between `kubectl run` and `kubectl apply -f`?

---

### ✏️ Exercise 2: Deployments & ReplicaSets

> **Goal:** Understand the Deployment → ReplicaSet → Pod hierarchy

**Step 1: Create a Deployment**
```yaml
# exercise2-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.24
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f exercise2-deployment.yaml
```

**Step 2: Observe the hierarchy**
```bash
# See all three levels
kubectl get deployments
kubectl get replicasets
kubectl get pods

# Notice the naming pattern:
# Deployment:  webapp
# ReplicaSet:  webapp-<hash>
# Pods:        webapp-<hash>-<random>

# Describe the deployment to see its events
kubectl describe deployment webapp
```

**Step 3: Test self-healing** 🔥
```bash
# Delete a pod and watch Kubernetes recreate it!
kubectl get pods
kubectl delete pod <paste-any-pod-name-here>

# Immediately watch:
kubectl get pods -w
# You'll see: a new pod being created automatically!
# Press Ctrl+C to stop watching
```

**Step 4: Scale the deployment**
```bash
# Scale up
kubectl scale deployment webapp --replicas=5
kubectl get pods  # Should show 5 pods

# Scale down
kubectl scale deployment webapp --replicas=2
kubectl get pods  # Should show 2 pods (3 terminating)
```

**Step 5: Update the image (Rolling Update)**
```bash
# Update nginx version
kubectl set image deployment/webapp webapp=nginx:1.25

# Watch the rolling update happen:
kubectl rollout status deployment webapp

# Check rollout history
kubectl rollout history deployment webapp

# 🔥 Rollback to previous version
kubectl rollout undo deployment webapp
kubectl rollout status deployment webapp
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise2-deployment.yaml
```

> **✅ Checkpoint:** Delete all pods of the deployment at once. What happens? Why?

---

### ✏️ Exercise 3: Services — Exposing Your App

> **Goal:** Understand ClusterIP, NodePort, and LoadBalancer services

**Step 1: Create a Deployment + ClusterIP Service**
```yaml
# exercise3-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: hashicorp/http-echo
        args:
        - "-text=Hello from Kubernetes! 🚀"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: hello-clusterip
spec:
  type: ClusterIP
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: hello-nodeport
spec:
  type: NodePort
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 5678
    nodePort: 30001
```

```bash
kubectl apply -f exercise3-service.yaml
```

**Step 2: Test ClusterIP (internal only)**
```bash
# ClusterIP is only accessible from INSIDE the cluster
kubectl get svc hello-clusterip

# Spin up a temporary pod to test:
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://hello-clusterip
# Expected output: Hello from Kubernetes! 🚀
```

**Step 3: Test NodePort (accessible from outside)**
```bash
kubectl get svc hello-nodeport
# Note the NodePort (30001)

# For Minikube:
minikube service hello-nodeport --url
# Open the URL in your browser!
```

**Step 4: Test load balancing**
```bash
# Run this multiple times and observe different pod IPs responding:
for i in $(seq 1 10); do
  kubectl run test-$i --image=busybox -it --rm -- wget -qO- http://hello-clusterip
done
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise3-service.yaml
```

> **✅ Checkpoint:** What's the difference between `port`, `targetPort`, and `nodePort`?

---

### ✏️ Exercise 4: Namespaces — Organizing Your Cluster

> **Goal:** Create isolated environments using namespaces

**Step 1: Create namespaces**
```yaml
# exercise4-namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```bash
kubectl apply -f exercise4-namespaces.yaml
kubectl get namespaces
```

**Step 2: Deploy same app in different namespaces**
```bash
# Deploy nginx in each namespace
kubectl create deployment nginx --image=nginx --replicas=2 -n development
kubectl create deployment nginx --image=nginx --replicas=2 -n staging
kubectl create deployment nginx --image=nginx --replicas=3 -n production

# See pods in each namespace
kubectl get pods -n development
kubectl get pods -n staging
kubectl get pods -n production

# See pods across ALL namespaces
kubectl get pods --all-namespaces
# or short form:
kubectl get pods -A
```

**Step 3: Set default namespace**
```bash
# Tired of typing -n every time?
kubectl config set-context --current --namespace=development
kubectl get pods   # Now shows development pods by default

# Switch back to default
kubectl config set-context --current --namespace=default
```

**Step 4: Cross-namespace communication**
```bash
# Create a service in development namespace
kubectl expose deployment nginx --port=80 -n development

# From a pod in staging, access development's service:
kubectl run test --image=busybox -n staging -it --rm -- \
  wget -qO- http://nginx.development.svc.cluster.local
# Format: <service-name>.<namespace>.svc.cluster.local
```

**🧹 Cleanup:**
```bash
kubectl delete namespace development staging production
```

> **✅ Checkpoint:** Can you explain the DNS format `service.namespace.svc.cluster.local`?

---

### ✏️ Exercise 5: Labels, Selectors & Annotations

> **Goal:** Master the key-value system that powers Kubernetes

**Step 1: Create labeled resources**
```yaml
# exercise5-labels.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-v1
  labels:
    app: myapp
    tier: frontend
    version: v1
    environment: production
spec:
  containers:
  - name: nginx
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: frontend-v2
  labels:
    app: myapp
    tier: frontend
    version: v2
    environment: staging
spec:
  containers:
  - name: nginx
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: backend-v1
  labels:
    app: myapp
    tier: backend
    version: v1
    environment: production
spec:
  containers:
  - name: nginx
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: database
  labels:
    app: myapp
    tier: database
    version: v1
    environment: production
  annotations:
    description: "Main production database"
    owner: "team-backend"
    backup-schedule: "daily-2am"
spec:
  containers:
  - name: postgres
    image: postgres:14
    env:
    - name: POSTGRES_PASSWORD
      value: "practice123"
```

```bash
kubectl apply -f exercise5-labels.yaml
```

**Step 2: Filter with selectors**
```bash
# Show all labels
kubectl get pods --show-labels

# Filter by single label
kubectl get pods -l tier=frontend
kubectl get pods -l environment=production

# Filter with multiple labels (AND logic)
kubectl get pods -l tier=frontend,environment=production

# NOT equal
kubectl get pods -l 'tier!=database'

# Set-based selector (IN)
kubectl get pods -l 'tier in (frontend, backend)'
kubectl get pods -l 'version in (v1, v2)'

# Check annotations
kubectl describe pod database | grep -A5 Annotations
```

**Step 3: Modify labels on the fly**
```bash
# Add a label
kubectl label pod frontend-v1 team=alpha
kubectl get pod frontend-v1 --show-labels

# Update a label
kubectl label pod frontend-v1 version=v1.1 --overwrite

# Remove a label (use minus sign)
kubectl label pod frontend-v1 team-
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise5-labels.yaml
```

> **✅ Checkpoint:** How do Services use labels to find which Pods to route traffic to?

---

## 🟡 Phase 2: Configuration & Storage (Exercises 6–10)

> Master ConfigMaps, Secrets, Volumes, and Resource Management

---

### ✏️ Exercise 6: ConfigMaps — Externalizing Configuration

> **Goal:** Inject configuration into Pods without hardcoding

**Step 1: Create ConfigMaps multiple ways**
```bash
# Method 1: From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080

# Method 2: From a file
echo "max_connections=100
timeout=30
log_level=info" > app.properties

kubectl create configmap file-config --from-file=app.properties

# View them
kubectl get configmaps
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

**Step 2: Use ConfigMap in a Pod**
```yaml
# exercise6-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  APP_COLOR: "blue"
  APP_MODE: "production"
  APP_VERSION: "2.0"
  nginx.conf: |
    server {
      listen 80;
      server_name localhost;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo Color=$APP_COLOR Mode=$APP_MODE Version=$APP_VERSION && sleep 3600"]
    # Method 1: Individual env vars
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: webapp-config
          key: APP_COLOR
    # Method 2: All keys as env vars
    envFrom:
    - configMapRef:
        name: webapp-config
    # Method 3: Mount as file
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: webapp-config
```

```bash
kubectl apply -f exercise6-configmap.yaml

# Check env vars
kubectl exec config-pod -- env | grep APP

# Check mounted files
kubectl exec config-pod -- ls /etc/config/
kubectl exec config-pod -- cat /etc/config/nginx.conf
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise6-configmap.yaml
kubectl delete configmap app-config file-config
```

> **✅ Checkpoint:** When would you use env vars vs volume mounts for ConfigMaps?

---

### ✏️ Exercise 7: Secrets — Handling Sensitive Data

> **Goal:** Safely manage passwords, API keys, and certificates

**Step 1: Create Secrets**
```bash
# Method 1: Imperative
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=supersecret123 \
  --from-literal=DB_HOST=postgres.default.svc.cluster.local

# View (notice base64 encoding)
kubectl get secret db-secret -o yaml
# ⚠️ base64 is NOT encryption — it's just encoding!

# Decode a value
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

**Step 2: Use Secrets in a Pod**
```yaml
# exercise7-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:                    # stringData = plain text (auto encoded)
  API_KEY: "my-super-secret-key-12345"
  JWT_SECRET: "jwt-token-secret-value"
data:                          # data = must be base64 encoded
  DB_PASSWORD: cGFzc3dvcmQxMjM=   # echo -n "password123" | base64
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo API_KEY=$API_KEY && echo JWT=$JWT_SECRET && sleep 3600"]
    envFrom:
    - secretRef:
        name: app-secret
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

```bash
kubectl apply -f exercise7-secrets.yaml

# Verify env vars
kubectl exec secret-pod -- env | grep -E "API_KEY|JWT|DB"

# Verify mounted files
kubectl exec secret-pod -- ls /etc/secrets/
kubectl exec secret-pod -- cat /etc/secrets/API_KEY
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise7-secrets.yaml
kubectl delete secret db-secret
```

> **✅ Checkpoint:** Why is base64 encoding NOT the same as encryption?

---

### ✏️ Exercise 8: Persistent Volumes & Claims

> **Goal:** Survive pod restarts with persistent storage

**Step 1: Create PV and PVC**
```yaml
# exercise8-storage.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: practice-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/data
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: practice-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo 'Written at: '$(date) >> /data/log.txt && cat /data/log.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: practice-pvc
```

```bash
kubectl apply -f exercise8-storage.yaml

# Check PV and PVC binding
kubectl get pv
kubectl get pvc

# Verify data is written
kubectl exec storage-pod -- cat /data/log.txt
```

**Step 2: Test persistence — delete pod and recreate**
```bash
# Delete the pod (but PVC stays!)
kubectl delete pod storage-pod

# Recreate the pod
kubectl apply -f exercise8-storage.yaml

# Check: old data should still be there + new entry!
kubectl exec storage-pod -- cat /data/log.txt
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise8-storage.yaml
kubectl delete pv practice-pv
```

> **✅ Checkpoint:** What happens to data when you delete the PVC with `Retain` vs `Delete` reclaim policy?

---

### ✏️ Exercise 9: Resource Requests & Limits

> **Goal:** Control how much CPU/memory your Pods can consume

```yaml
# exercise9-resources.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: stress-test
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "1", "--vm", "1", "--vm-bytes", "128M", "--timeout", "60s"]
    resources:
      requests:           # Guaranteed minimum
        cpu: 100m         # 100 millicores = 0.1 CPU
        memory: 64Mi      # 64 Megabytes
      limits:             # Maximum allowed
        cpu: 200m         # 200 millicores = 0.2 CPU
        memory: 256Mi     # 256 Megabytes
---
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
  - name: memory-hog
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "512M", "--timeout", "600s"]
    resources:
      requests:
        memory: 64Mi
      limits:
        memory: 128Mi     # Pod will be OOMKilled!
```

```bash
kubectl apply -f exercise9-resources.yaml

# Watch resource-demo complete successfully
kubectl get pod resource-demo -w

# Watch oom-demo get killed
kubectl get pod oom-demo -w
# Status will change to OOMKilled!

# Check why it was killed
kubectl describe pod oom-demo | grep -A5 "Last State"

# Monitor resource usage (requires metrics-server)
kubectl top pods
kubectl top nodes
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise9-resources.yaml
```

> **✅ Checkpoint:** What's the difference between requests and limits? What happens when a pod exceeds each?

---

### ✏️ Exercise 10: Multi-Container Pods (Sidecar Pattern)

> **Goal:** Run multiple containers in a single Pod that share resources

```yaml
# exercise10-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
  # Main application container — writes logs
  - name: app
    image: busybox
    command: ["sh", "-c"]
    args:
    - |
      i=0
      while true; do
        echo "$(date) - Log entry $i: Application is running" >> /var/log/app.log
        i=$((i+1))
        sleep 5
      done
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

  # Sidecar container — reads and forwards logs
  - name: log-agent
    image: busybox
    command: ["sh", "-c"]
    args:
    - |
      echo "Log agent started, tailing /var/log/app.log..."
      tail -f /var/log/app.log
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

  volumes:
  - name: shared-logs
    emptyDir: {}
```

```bash
kubectl apply -f exercise10-sidecar.yaml

# Wait a few seconds then check the log agent's output:
kubectl logs sidecar-demo -c log-agent

# Check the app container's logs:
kubectl logs sidecar-demo -c app

# Exec into app container
kubectl exec -it sidecar-demo -c app -- cat /var/log/app.log

# Notice: both containers share the same volume!
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise10-sidecar.yaml
```

> **✅ Checkpoint:** Why do sidecar containers share the same network and storage as the main container?

---

## 🔴 Phase 3: Advanced Features (Exercises 11–16)

> Probes, Ingress, Jobs, RBAC, Network Policies, and Autoscaling

---

### ✏️ Exercise 11: Liveness & Readiness Probes

> **Goal:** Make Kubernetes automatically detect unhealthy pods

```yaml
# exercise11-probes.yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80

    # Is the container alive? If not → restart it
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3

    # Is the container ready to accept traffic? If not → remove from Service
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5

    # Is the container started? (for slow-starting apps)
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 10
---
# A pod that will FAIL its liveness probe
apiVersion: v1
kind: Pod
metadata:
  name: unhealthy-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 600"]

    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5
      periodSeconds: 5
```

```bash
kubectl apply -f exercise11-probes.yaml

# Watch healthy pod — stays running
kubectl get pod probe-demo -w

# Watch unhealthy pod — gets restarted after 30 seconds!
kubectl get pod unhealthy-pod -w
# After ~35 seconds: RESTARTS will increment

# Check events
kubectl describe pod unhealthy-pod | grep -A10 Events
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise11-probes.yaml
```

> **✅ Checkpoint:** What happens if readiness probe fails vs liveness probe fails?

---

### ✏️ Exercise 12: Jobs & CronJobs

> **Goal:** Run one-time and scheduled tasks in Kubernetes

```yaml
# exercise12-jobs.yaml
# One-time Job
apiVersion: batch/v1
kind: Job
metadata:
  name: math-job
spec:
  completions: 5        # Run 5 times total
  parallelism: 2        # Run 2 at a time
  backoffLimit: 3       # Retry up to 3 times on failure
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: math
        image: busybox
        command: ["sh", "-c", "echo 'Computing...' && echo $((RANDOM % 100 + 1)) && sleep 5"]
---
# Scheduled CronJob (runs every minute)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: heartbeat
spec:
  schedule: "*/1 * * * *"     # Every minute
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: heartbeat
            image: busybox
            command: ["sh", "-c", "echo '💓 Heartbeat at: '$(date)"]
```

```bash
kubectl apply -f exercise12-jobs.yaml

# Watch the Job progress
kubectl get jobs -w
kubectl get pods --show-labels

# Check individual pod outputs
kubectl logs job/math-job

# Watch CronJob create new Jobs every minute
kubectl get cronjobs
kubectl get jobs -w    # New jobs appear every minute!

# Check CronJob's pod logs
kubectl get pods | grep heartbeat
kubectl logs <heartbeat-pod-name>
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise12-jobs.yaml
```

> **✅ Checkpoint:** What's the difference between `completions` and `parallelism`?

---

### ✏️ Exercise 13: Ingress Controller

> **Goal:** Route external traffic to services using hostnames and paths

**Prerequisite:**
```bash
# Enable ingress on Minikube
minikube addons enable ingress

# Wait for ingress controller to be ready
kubectl get pods -n ingress-nginx -w
```

```yaml
# exercise13-ingress.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      version: v1
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=🟢 Version 1"]
        ports:
        - containerPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      version: v2
  template:
    metadata:
      labels:
        app: web
        version: v2
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=🔵 Version 2"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app-v1-svc
spec:
  selector:
    app: web
    version: v1
  ports:
  - port: 80
    targetPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app-v2-svc
spec:
  selector:
    app: web
    version: v2
  ports:
  - port: 80
    targetPort: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: app-v1-svc
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: app-v2-svc
            port:
              number: 80
```

```bash
kubectl apply -f exercise13-ingress.yaml

# Get minikube IP
minikube ip

# Add to hosts file (run as Administrator!):
# Add this line to C:\Windows\System32\drivers\etc\hosts
# <minikube-ip>  myapp.local

# Test
curl http://myapp.local/v1    # 🟢 Version 1
curl http://myapp.local/v2    # 🔵 Version 2
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise13-ingress.yaml
```

> **✅ Checkpoint:** How is Ingress different from a NodePort Service?

---

### ✏️ Exercise 14: RBAC — Role-Based Access Control

> **Goal:** Control who can do what in your cluster

```yaml
# exercise14-rbac.yaml
# Create a namespace for this exercise
apiVersion: v1
kind: Namespace
metadata:
  name: rbac-practice
---
# Create a ServiceAccount (representing a user/app)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dev-user
  namespace: rbac-practice
---
# Role: defines WHAT actions are allowed
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-practice
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---
# RoleBinding: binds the Role to the ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: rbac-practice
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: rbac-practice
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
---
# Create a test pod
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: rbac-practice
spec:
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f exercise14-rbac.yaml

# Test: Can dev-user list pods? ✅ YES
kubectl auth can-i list pods -n rbac-practice --as=system:serviceaccount:rbac-practice:dev-user

# Test: Can dev-user delete pods? ❌ NO
kubectl auth can-i delete pods -n rbac-practice --as=system:serviceaccount:rbac-practice:dev-user

# Test: Can dev-user create deployments? ❌ NO
kubectl auth can-i create deployments -n rbac-practice --as=system:serviceaccount:rbac-practice:dev-user

# Test: Can dev-user access other namespaces? ❌ NO
kubectl auth can-i list pods -n default --as=system:serviceaccount:rbac-practice:dev-user
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise14-rbac.yaml
kubectl delete namespace rbac-practice
```

> **✅ Checkpoint:** What's the difference between Role/RoleBinding and ClusterRole/ClusterRoleBinding?

---

### ✏️ Exercise 15: Horizontal Pod Autoscaler (HPA)

> **Goal:** Auto-scale pods based on CPU utilization

**Prerequisite:**
```bash
minikube addons enable metrics-server
# Wait 1-2 minutes for metrics to be available
kubectl top nodes
```

```yaml
# exercise15-hpa.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    app: php-apache
  ports:
  - port: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

```bash
kubectl apply -f exercise15-hpa.yaml

# Terminal 1: Watch the HPA
kubectl get hpa php-apache-hpa -w

# Terminal 2: Generate load!
kubectl run -it --rm load-generator --image=busybox -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

# Watch pods scale up in Terminal 1!
# After a minute, you should see replicas increase

# Stop the load generator (Ctrl+C) and watch pods scale back down
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise15-hpa.yaml
```

> **✅ Checkpoint:** Why does HPA need a metrics-server? What metrics can you scale on besides CPU?

---

### ✏️ Exercise 16: Network Policies

> **Goal:** Control traffic flow between pods

```yaml
# exercise16-networkpolicy.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: netpol-demo
  labels:
    purpose: network-policy-demo
---
# Frontend pod
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: netpol-demo
  labels:
    role: frontend
spec:
  containers:
  - name: nginx
    image: nginx
---
# Backend pod
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: netpol-demo
  labels:
    role: backend
spec:
  containers:
  - name: nginx
    image: nginx
---
# Database pod
apiVersion: v1
kind: Pod
metadata:
  name: database
  namespace: netpol-demo
  labels:
    role: database
spec:
  containers:
  - name: nginx
    image: nginx
---
# Services for each
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: netpol-demo
spec:
  selector:
    role: backend
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: database-svc
  namespace: netpol-demo
spec:
  selector:
    role: database
  ports:
  - port: 80
---
# Network Policy: Only backend can talk to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      role: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - port: 80
```

```bash
kubectl apply -f exercise16-networkpolicy.yaml

# Wait for pods to be ready
kubectl get pods -n netpol-demo -w

# Test: Backend → Database ✅ (should work)
kubectl exec -n netpol-demo backend -- wget -qO- --timeout=3 http://database-svc

# Test: Frontend → Database ❌ (should be blocked!)
kubectl exec -n netpol-demo frontend -- wget -qO- --timeout=3 http://database-svc
# This should timeout!

# Note: NetworkPolicy requires a CNI plugin that supports it
# (e.g., Calico, Cilium). Minikube's default might not enforce it.
# Install Calico: minikube start --cni=calico
```

**🧹 Cleanup:**
```bash
kubectl delete namespace netpol-demo
```

> **✅ Checkpoint:** What happens when no NetworkPolicy exists? (default behavior)

---

## 🚀 Phase 4: Real-World Challenges (Exercises 17–20)

> Put it all together with debugging exercises and a final project

---

### ✏️ Exercise 17: 🔥 Debugging Challenge — Fix the Broken Pod

> **Goal:** Practice real-world debugging skills

```yaml
# exercise17-debug.yaml — These have INTENTIONAL ERRORS!
# Your job: apply them, find the errors, and fix them!

# Challenge 1: ImagePullBackOff
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod-1
spec:
  containers:
  - name: app
    image: nginx:doesnotexist999

---
# Challenge 2: CrashLoopBackOff
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod-2
spec:
  containers:
  - name: app
    image: busybox
    command: ["exit", "1"]

---
# Challenge 3: Pending (impossible resource request)
apiVersion: v1
kind: Pod
metadata:
  name: broken-pod-3
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100"        # 100 CPUs! No node has this
        memory: "1000Gi"

---
# Challenge 4: Service not routing traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp        # Pod label = "myapp"
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: broken-service
spec:
  selector:
    app: wronglabel       # ❌ Selector doesn't match pod label!
  ports:
  - port: 80
    targetPort: 80
```

**Debugging Commands Cheatsheet:**
```bash
kubectl apply -f exercise17-debug.yaml

# Step 1: Get overview
kubectl get pods

# Step 2: Describe failed pods
kubectl describe pod broken-pod-1
kubectl describe pod broken-pod-2
kubectl describe pod broken-pod-3

# Step 3: Check logs
kubectl logs broken-pod-2

# Step 4: Check events
kubectl get events --sort-by='.lastTimestamp'

# Step 5: Debug service connectivity
kubectl describe svc broken-service
kubectl get endpoints broken-service  # Empty endpoints = wrong selector!

# YOUR TASK: Fix each issue and re-apply!
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise17-debug.yaml
```

---

### ✏️ Exercise 18: Rolling Updates & Rollbacks

> **Goal:** Deploy updates with zero downtime

```yaml
# exercise18-rolling.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-app
  annotations:
    kubernetes.io/change-cause: "Initial deployment with v1.24"
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2           # Max extra pods during update
      maxUnavailable: 1     # Max pods that can be down
  selector:
    matchLabels:
      app: rolling
  template:
    metadata:
      labels:
        app: rolling
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: rolling-svc
spec:
  type: NodePort
  selector:
    app: rolling
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
kubectl apply -f exercise18-rolling.yaml

# Watch the initial deployment
kubectl get pods -l app=rolling -w

# ─── Update 1: Good update ───
kubectl set image deployment/rolling-app nginx=nginx:1.25 --record
kubectl rollout status deployment rolling-app
# Watch pods being replaced one by one!

# ─── Update 2: BAD update (broken image) ───
kubectl set image deployment/rolling-app nginx=nginx:broken-image --record
kubectl rollout status deployment rolling-app
# This will hang — some pods can't start!

# Check status
kubectl get pods -l app=rolling
# You'll see ImagePullBackOff errors

# ─── Rollback! ───
kubectl rollout history deployment rolling-app     # See all versions
kubectl rollout undo deployment rolling-app        # Go back to last working version
kubectl rollout status deployment rolling-app      # Verify recovery

# Rollback to a specific revision
kubectl rollout undo deployment rolling-app --to-revision=1
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise18-rolling.yaml
```

> **✅ Checkpoint:** What do `maxSurge` and `maxUnavailable` control? What happens if both are 0?

---

### ✏️ Exercise 19: Init Containers & Pod Lifecycle

> **Goal:** Understand how Pods start up with initialization steps

```yaml
# exercise19-init.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  # Init containers run BEFORE app containers, in order
  initContainers:
  - name: init-check-service
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "⏳ Init Container 1: Checking if service exists..."
      until nslookup myservice.default.svc.cluster.local; do
        echo "Waiting for myservice..."
        sleep 2
      done
      echo "✅ Service found!"

  - name: init-create-config
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "⏳ Init Container 2: Creating config file..."
      echo '{"initialized": true, "timestamp": "'$(date)'"}' > /config/init.json
      echo "✅ Config created!"
    volumeMounts:
    - name: config
      mountPath: /config

  containers:
  - name: app
    image: busybox
    command: ['sh', '-c']
    args:
    - |
      echo "🚀 Main app started!"
      echo "Config contents:"
      cat /config/init.json
      sleep 3600
    volumeMounts:
    - name: config
      mountPath: /config

  volumes:
  - name: config
    emptyDir: {}
```

```bash
kubectl apply -f exercise19-init.yaml

# Watch the pod — it will be stuck in Init:0/2
kubectl get pod init-demo -w

# Check what init container is waiting for:
kubectl logs init-demo -c init-check-service

# Create the service it's waiting for:
kubectl create service clusterip myservice --tcp=80:80

# Now watch the pod progress through init containers → Running!
kubectl get pod init-demo -w

# Check the config created by init container:
kubectl exec init-demo -- cat /config/init.json
```

**🧹 Cleanup:**
```bash
kubectl delete -f exercise19-init.yaml
kubectl delete svc myservice
```

> **✅ Checkpoint:** In what order do init containers run? What if one fails?

---

### ✏️ Exercise 20: 🏆 Final Project — Deploy a Complete Application Stack

> **Goal:** Combine everything you've learned into one deployment!

**Architecture:**
```
┌──────────────────────────────────────────────────────────────────┐
│                    📦 COMPLETE APP STACK                          │
│                                                                   │
│   Internet ──► Ingress ──┬──► /        ──► Frontend Service      │
│                          └──► /api     ──► Backend Service       │
│                                               │                   │
│                                               ▼                   │
│                                        Redis Service              │
│                                               │                   │
│                                               ▼                   │
│                                         Redis Pod                 │
│                                    (PVC for persistence)          │
│                                                                   │
│   Features used:                                                  │
│   ✅ Namespace          ✅ Deployments       ✅ Services          │
│   ✅ ConfigMap          ✅ Secrets           ✅ PVC               │
│   ✅ Ingress            ✅ HPA               ✅ Probes            │
│   ✅ Resource Limits    ✅ Labels            ✅ Init Container    │
└──────────────────────────────────────────────────────────────────┘
```

```yaml
# exercise20-final-project.yaml
# ═══════════════════════════════════════════════
# PART 1: Foundation (Namespace + Config + Secrets)
# ═══════════════════════════════════════════════

apiVersion: v1
kind: Namespace
metadata:
  name: fullstack-app
  labels:
    project: fullstack
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: fullstack-app
data:
  APP_ENV: "production"
  REDIS_HOST: "redis-svc"
  REDIS_PORT: "6379"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: fullstack-app
type: Opaque
stringData:
  REDIS_PASSWORD: "redis-secure-pass-123"
  API_SECRET_KEY: "my-super-secret-api-key"

---
# ═══════════════════════════════════════════════
# PART 2: Redis (Database Layer)
# ═══════════════════════════════════════════════

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: fullstack-app
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: fullstack-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
      tier: database
  template:
    metadata:
      labels:
        app: redis
        tier: database
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        livenessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 3
          periodSeconds: 5
        volumeMounts:
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-data
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: fullstack-app
spec:
  selector:
    app: redis
    tier: database
  ports:
  - port: 6379
    targetPort: 6379

---
# ═══════════════════════════════════════════════
# PART 3: Backend API
# ═══════════════════════════════════════════════

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: fullstack-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: api
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      initContainers:
      - name: wait-for-redis
        image: busybox
        command: ['sh', '-c']
        args:
        - |
          echo "Waiting for Redis..."
          until nc -z redis-svc 6379; do
            echo "Redis not ready, retrying in 2s..."
            sleep 2
          done
          echo "Redis is ready!"
      containers:
      - name: backend
        image: hashicorp/http-echo
        args: ["-text=🟢 Backend API - Connected to Redis"]
        ports:
        - containerPort: 5678
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 300m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /
            port: 5678
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 5678
          initialDelaySeconds: 3
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: fullstack-app
spec:
  selector:
    app: backend
    tier: api
  ports:
  - port: 80
    targetPort: 5678

---
# ═══════════════════════════════════════════════
# PART 4: Frontend
# ═══════════════════════════════════════════════

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: fullstack-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: frontend
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 200m
            memory: 128Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: fullstack-app
spec:
  selector:
    app: frontend
    tier: web
  ports:
  - port: 80
    targetPort: 80

---
# ═══════════════════════════════════════════════
# PART 5: Ingress (External Access)
# ═══════════════════════════════════════════════

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: fullstack-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: fullstack.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 80

---
# ═══════════════════════════════════════════════
# PART 6: Autoscaling
# ═══════════════════════════════════════════════

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: fullstack-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: fullstack-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Deploy & Verify:**
```bash
# Deploy everything
kubectl apply -f exercise20-final-project.yaml

# Watch everything come up
kubectl get all -n fullstack-app -w

# Verify each layer:
# 1. Check Redis
kubectl get pods -n fullstack-app -l tier=database
kubectl exec -n fullstack-app deploy/redis -- redis-cli ping

# 2. Check Backend (should show init container completed)
kubectl get pods -n fullstack-app -l tier=api
kubectl logs -n fullstack-app deploy/backend -c wait-for-redis

# 3. Check Frontend
kubectl get pods -n fullstack-app -l tier=web

# 4. Check Services
kubectl get svc -n fullstack-app

# 5. Check Ingress
kubectl get ingress -n fullstack-app

# 6. Check HPA
kubectl get hpa -n fullstack-app

# 7. Check all ConfigMaps and Secrets
kubectl get cm,secrets -n fullstack-app

# 8. Verify env vars in backend
kubectl exec -n fullstack-app deploy/backend -- env | grep -E "APP_ENV|REDIS"

# 9. Test connectivity
kubectl run test -n fullstack-app --image=busybox -it --rm -- wget -qO- http://frontend-svc
kubectl run test -n fullstack-app --image=busybox -it --rm -- wget -qO- http://backend-svc

# For Minikube — add to hosts file and test:
# <minikube-ip>  fullstack.local
```

**🧹 Cleanup:**
```bash
kubectl delete namespace fullstack-app
```

---

## 📊 Progress Tracker

Use this checklist to track your progress:

```
Phase 1: Core Basics
  [ ] Exercise 1:  Your First Pod
  [ ] Exercise 2:  Deployments & ReplicaSets
  [ ] Exercise 3:  Services (ClusterIP, NodePort)
  [ ] Exercise 4:  Namespaces
  [ ] Exercise 5:  Labels, Selectors & Annotations

Phase 2: Configuration & Storage
  [ ] Exercise 6:  ConfigMaps
  [ ] Exercise 7:  Secrets
  [ ] Exercise 8:  Persistent Volumes & Claims
  [ ] Exercise 9:  Resource Requests & Limits
  [ ] Exercise 10: Multi-Container Pods (Sidecar)

Phase 3: Advanced Features
  [ ] Exercise 11: Liveness & Readiness Probes
  [ ] Exercise 12: Jobs & CronJobs
  [ ] Exercise 13: Ingress Controller
  [ ] Exercise 14: RBAC
  [ ] Exercise 15: Horizontal Pod Autoscaler
  [ ] Exercise 16: Network Policies

Phase 4: Real-World Challenges
  [ ] Exercise 17: 🔥 Debugging Challenge
  [ ] Exercise 18: Rolling Updates & Rollbacks
  [ ] Exercise 19: Init Containers & Pod Lifecycle
  [ ] Exercise 20: 🏆 Final Project — Full Stack
```

---

## 💡 Pro Tips for Practice

1. **Write YAML from scratch** — Don't copy-paste. Muscle memory matters.
2. **Use `--dry-run=client -o yaml`** — Generate templates when stuck.
3. **Break things deliberately** — Change labels, delete pods, corrupt images.
4. **Read error messages** — `kubectl describe` and `kubectl get events` are your best friends.
5. **Time yourself** — Try to complete each exercise faster each time.
6. **Explain out loud** — If you can explain what each YAML field does, you understand it.

---

## 🌐 Additional Practice Resources

| Resource | URL | Type |
|----------|-----|------|
| Killercoda | https://killercoda.com | Free browser labs |
| Play with K8s | https://labs.play-with-k8s.com | Free playground |
| KodeKloud | https://kodekloud.com | Guided exercises |
| Kubernetes Docs | https://kubernetes.io/docs/tutorials/ | Official tutorials |
| CKAD Exercises | https://github.com/dgkanatsios/CKAD-exercises | Certification prep |

---

**⬅️ Previous: [14 - Cheat Sheet](./14-cheatsheet.md)** | **🏠 [Index](./00-kubernetes-index.md)**
