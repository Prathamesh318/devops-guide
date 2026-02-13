# 🌐 Chapter 7: Networking & Services

> **"Services are the stable front door to your ephemeral pods"**

---

## 🎯 Why Services?

```
┌────────────────────────────────────────────────────────────────┐
│                    THE PROBLEM                                  │
│                                                                 │
│   Pod IPs are ephemeral:                                       │
│   • Pod dies → New pod gets NEW IP                             │
│   • Can't hardcode pod IPs                                     │
│   • Need stable endpoint for communication                     │
│                                                                 │
│   ┌─────────┐        ┌─────────┐                              │
│   │ nginx-1 │ ──X──► │ app pod │  (Old IP, pod restarted)    │
│   │10.0.0.5 │        │10.0.0.8 │                              │
│   └─────────┘        └────┘                                   │
│                          ↓ restart                             │
│                      ┌─────────┐                              │
│                      │ app pod │  (New IP!)                   │
│                      │10.0.0.9 │                              │
│                      └─────────┘                              │
│                                                                 │
│   THE SOLUTION: Services provide stable endpoint!              │
└────────────────────────────────────────────────────────────────┘
```

---

## 📦 Service Types

| Type | Access From | Use Case |
|------|-------------|----------|
| **ClusterIP** | Inside cluster only | Internal services |
| **NodePort** | External via node IP:port | Development, testing |
| **LoadBalancer** | External via cloud LB | Production in cloud |
| **ExternalName** | DNS alias | External services |

```
┌──────────────────────────────────────────────────────────────────┐
│                    SERVICE TYPES                                  │
│                                                                   │
│   ClusterIP (default)                                            │
│   ┌───────────┐                                                  │
│   │  Service  │ ◄── Only accessible inside cluster             │
│   │ 10.96.0.1 │                                                  │
│   └─────┬─────┘                                                  │
│         └──► Pods                                                │
│                                                                   │
│   NodePort                                                        │
│   ┌───────────┐                                                  │
│   │   Node    │ :30080 ◄── External access via node IP         │
│   │ 192.168.x │                                                  │
│   └─────┬─────┘                                                  │
│         └──► Service ──► Pods                                    │
│                                                                   │
│   LoadBalancer                                                    │
│   ┌───────────┐                                                  │
│   │ Cloud LB  │ ◄── External IP from cloud provider            │
│   │ 34.1.2.3  │                                                  │
│   └─────┬─────┘                                                  │
│         └──► Service ──► Pods                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔵 ClusterIP Service

> **Default type - internal cluster communication**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # default, can omit
  selector:
    app: backend
  ports:
  - port: 80          # Service port
    targetPort: 8080  # Container port
```

### Access ClusterIP
```bash
# From inside cluster
curl http://backend-service:80
curl http://backend-service.default.svc.cluster.local:80
```

---

## 🟠 NodePort Service

> **Exposes service on each node's IP at a static port**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80          # Service port
    targetPort: 8080  # Container port  
    nodePort: 30080   # External port (30000-32767)
```

### Access NodePort
```bash
# From outside cluster
curl http://<NODE_IP>:30080
```

---

## 🟢 LoadBalancer Service

> **Creates cloud load balancer with external IP**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

### Access LoadBalancer
```bash
kubectl get svc web-lb
# EXTERNAL-IP shows the load balancer IP
curl http://<EXTERNAL-IP>:80
```

---

## 🔗 ExternalName Service

> **Maps service to external DNS name**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.external.com
```

---

## 🎯 Headless Service

> **Direct pod discovery without load balancing (for StatefulSets)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None  # This makes it headless!
  selector:
    app: mysql
  ports:
  - port: 3306
```

### DNS Resolution
```
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
```

---

## 🚪 Ingress

> **HTTP/HTTPS routing to services based on host/path**

```
┌────────────────────────────────────────────────────────────────┐
│                       INGRESS CONCEPT                           │
│                                                                 │
│   Internet                                                      │
│       │                                                         │
│       ▼                                                         │
│   ┌───────────────────────────────────────────────────────┐    │
│   │                  INGRESS CONTROLLER                    │    │
│   │              (nginx, traefik, etc.)                   │    │
│   └────────────────────┬──────────────────────────────────┘    │
│                        │                                        │
│     ┌──────────────────┼──────────────────┐                    │
│     │                  │                  │                    │
│     ▼                  ▼                  ▼                    │
│  /api/*            /web/*            /admin/*                  │
│     │                  │                  │                    │
│     ▼                  ▼                  ▼                    │
│  ┌──────┐          ┌──────┐          ┌──────┐                 │
│  │ api  │          │ web  │          │admin │                 │
│  │ svc  │          │ svc  │          │ svc  │                 │
│  └──────┘          └──────┘          └──────┘                 │
└────────────────────────────────────────────────────────────────┘
```

### Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
```

### Enable Ingress (Minikube)
```bash
minikube addons enable ingress
kubectl get ingress
```

---

## 🔌 Service Commands

```bash
# List services
kubectl get services
kubectl get svc  # short

# Create service imperatively
kubectl expose deployment nginx --port=80 --type=NodePort

# Describe service
kubectl describe svc my-service

# Get endpoints
kubectl get endpoints

# Delete service
kubectl delete svc my-service
```

---

## 📊 Service DNS

```
┌────────────────────────────────────────────────────────────────┐
│                    SERVICE DNS PATTERNS                         │
│                                                                 │
│   Short name (same namespace):                                 │
│   my-service                                                    │
│                                                                 │
│   With namespace:                                               │
│   my-service.production                                         │
│                                                                 │
│   Full FQDN:                                                    │
│   my-service.production.svc.cluster.local                      │
│                                                                 │
│   Format: <service>.<namespace>.svc.cluster.local              │
└────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Memory Shortcuts

### Service Types: **"CNLE"**
```
C = ClusterIP (internal, default)
N = NodePort (node IP + port)
L = LoadBalancer (cloud LB)
E = ExternalName (DNS alias)
```

### Port Mapping Flow
```
External → NodePort(30080) → Service Port(80) → TargetPort(8080) → Container
```

---

**Next: [08 - Configuration & Secrets](./08-configuration-secrets.md)** ➡️
