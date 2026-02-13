# 🛠️ Chapter 13: Hands-on Projects

> **"Theory without practice is empty. Let's build real applications!"**

---

## 📋 Project 1: Django Notes App

> **Deploy a full-stack web application with PostgreSQL**

### Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                  DJANGO NOTES APP                               │
│                                                                 │
│   Internet                                                      │
│       │                                                         │
│       ▼                                                         │
│   ┌─────────┐                                                  │
│   │ Ingress │                                                  │
│   └────┬────┘                                                  │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐               │
│   │ Service │ ───► │ Django  │ ───► │  PG     │               │
│   │ (LB)    │      │  Pods   │      │ Service │               │
│   └─────────┘      └─────────┘      └────┬────┘               │
│                                          │                     │
│                                          ▼                     │
│                                    ┌─────────┐                │
│                                    │Postgres │                │
│                                    │   Pod   │                │
│                                    └─────────┘                │
└────────────────────────────────────────────────────────────────┘
```

### Step 1: Create Namespace

```yaml
# 01-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: notes-app
```

### Step 2: PostgreSQL Deployment

```yaml
# 02-postgres.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: notes-app
type: Opaque
stringData:
  POSTGRES_DB: notes
  POSTGRES_USER: admin
  POSTGRES_PASSWORD: secretpassword
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: notes-app
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
  name: postgres
  namespace: notes-app
spec:
  replicas: 1
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
        image: postgres:14
        ports:
        - containerPort: 5432
        envFrom:
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: notes-app
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

### Step 3: Django Application

```yaml
# 03-django.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
  namespace: notes-app
data:
  DEBUG: "False"
  ALLOWED_HOSTS: "*"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
  namespace: notes-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      containers:
      - name: django
        image: your-django-image:latest
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: django-config
        - secretRef:
            name: postgres-secret
        env:
        - name: DATABASE_HOST
          value: postgres-service
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: django-service
  namespace: notes-app
spec:
  type: LoadBalancer
  selector:
    app: django
  ports:
  - port: 80
    targetPort: 8000
```

### Deploy Commands

```bash
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-postgres.yaml
kubectl apply -f 03-django.yaml

# Check status
kubectl get all -n notes-app

# Access app
kubectl get svc -n notes-app
```

---

## 📋 Project 2: Nginx with HPA

> **Auto-scaling web server based on CPU usage**

```yaml
# nginx-hpa.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-scalable
spec:
  replicas: 2
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
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-scalable
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

### Load Test

```bash
# Apply deployment
kubectl apply -f nginx-hpa.yaml

# Watch HPA
kubectl get hpa -w

# Generate load (in another terminal)
kubectl run -it --rm load-test --image=busybox -- /bin/sh
# Inside pod:
while true; do wget -q -O- http://nginx-service; done
```

---

## 📋 Project 3: Multi-tier App with Ingress

```yaml
# multi-tier.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args:
        - "-text=API Response"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
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
            name: api-svc
            port:
              number: 80
```

---

## 🔧 Useful Practice Commands

```bash
# Quick deployment test
kubectl create deployment test --image=nginx --replicas=3
kubectl expose deployment test --port=80 --type=NodePort

# Debug with busybox
kubectl run debug --image=busybox -it --rm -- /bin/sh

# Test DNS
kubectl run test --image=busybox -it --rm -- nslookup kubernetes

# Check resources
kubectl get all --all-namespaces
```

---

**Next: [14 - Cheat Sheet](./14-cheatsheet.md)** ➡️
