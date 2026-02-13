# ⚙️ Chapter 8: Configuration & Secrets

> **"Separate configuration from code for flexible, secure deployments"**

---

## 🎯 Why Externalize Config?

```
┌────────────────────────────────────────────────────────────────┐
│                    12-FACTOR APP PRINCIPLE                      │
│                                                                 │
│   ❌ Bad: Hardcoded config in container                        │
│      docker build → image with DB_HOST=prod-db                 │
│      (Need to rebuild for each environment!)                   │
│                                                                 │
│   ✅ Good: External config injected at runtime                 │
│      Same image → different ConfigMaps per environment         │
│      Dev: DB_HOST=dev-db                                       │
│      Prod: DB_HOST=prod-db                                     │
└────────────────────────────────────────────────────────────────┘
```

---

## 📋 ConfigMaps

> **Store non-sensitive configuration data**

### Creating ConfigMaps

```bash
# From literal values
kubectl create configmap my-config \
  --from-literal=DB_HOST=localhost \
  --from-literal=DB_PORT=5432

# From file
kubectl create configmap app-config \
  --from-file=config.properties

# From directory
kubectl create configmap configs \
  --from-file=./config-dir/
```

### ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple key-value
  DB_HOST: "database.example.com"
  DB_PORT: "5432"
  LOG_LEVEL: "info"
  
  # Multi-line file content
  app.properties: |
    server.port=8080
    spring.profiles.active=production
    logging.level.root=INFO
```

### Using ConfigMap in Pods

#### Method 1: Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    env:
    # Single key
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DB_HOST
    # All keys as env vars
    envFrom:
    - configMapRef:
        name: app-config
```

#### Method 2: Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

### ConfigMap Commands

```bash
kubectl get configmaps
kubectl get cm  # short
kubectl describe cm app-config
kubectl delete cm app-config
```

---

## 🔐 Secrets

> **Store sensitive data (passwords, tokens, keys)**

⚠️ **Important:** Secrets are base64 encoded, NOT encrypted by default!

### Secret Types

| Type | Use Case |
|------|----------|
| **Opaque** | Generic secrets (default) |
| **kubernetes.io/tls** | TLS certificates |
| **kubernetes.io/dockerconfigjson** | Docker registry auth |
| **kubernetes.io/basic-auth** | Username/password |
| **kubernetes.io/ssh-auth** | SSH private key |

### Creating Secrets

```bash
# From literal values
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secretpass

# From file
kubectl create secret generic ssh-key \
  --from-file=ssh-privatekey=~/.ssh/id_rsa

# TLS secret
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=user \
  --docker-password=pass
```

### Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Values must be base64 encoded!
  username: YWRtaW4=      # echo -n "admin" | base64
  password: c2VjcmV0cGFzcw==  # echo -n "secretpass" | base64
---
# Or use stringData (plain text, auto-encoded)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret-plain
type: Opaque
stringData:
  username: admin
  password: secretpass
```

### Using Secrets in Pods

#### Method 1: Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

#### Method 2: Volume Mount

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
```

### Secret Commands

```bash
kubectl get secrets
kubectl describe secret db-secret
kubectl get secret db-secret -o yaml

# Decode secret value
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

---

## 📊 ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Data** | Non-sensitive | Sensitive |
| **Encoding** | Plain text | Base64 |
| **Size limit** | 1 MB | 1 MB |
| **Encryption** | No | Optional (at rest) |
| **Use case** | URLs, ports, configs | Passwords, tokens |

---

## 🔄 Updating Configs

### ConfigMap/Secret Updates

```
┌────────────────────────────────────────────────────────────────┐
│               UPDATE BEHAVIOR                                   │
│                                                                 │
│   Environment Variables:                                        │
│   • Pod restart required to pick up changes                    │
│   • Existing pods keep old values                              │
│                                                                 │
│   Volume Mounts:                                                │
│   • Auto-update within ~1 minute (kubelet sync)                │
│   • No pod restart needed                                      │
│   • App must watch for file changes                            │
└────────────────────────────────────────────────────────────────┘
```

### Force Pod Restart After Config Update

```bash
# Rolling restart deployment
kubectl rollout restart deployment my-app
```

---

## 🛡️ Best Practices

```
┌────────────────────────────────────────────────────────────────┐
│               CONFIGURATION BEST PRACTICES                      │
│                                                                 │
│   ConfigMaps:                                                   │
│   ✅ Use for non-sensitive config                              │
│   ✅ Version configs (app-config-v1, app-config-v2)           │
│   ✅ Use immutable ConfigMaps for stability                    │
│                                                                 │
│   Secrets:                                                      │
│   ✅ Enable encryption at rest (etcd)                          │
│   ✅ Use external secret managers (Vault, AWS Secrets)         │
│   ✅ Limit RBAC access to secrets                              │
│   ✅ Rotate secrets regularly                                  │
│   ❌ Never commit secrets to Git                               │
│   ❌ Never log secret values                                   │
└────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Memory Shortcuts

### Config Injection: **"EV"**
```
E = Environment variables (valueFrom)
V = Volume mounts (files)
```

### ConfigMap vs Secret: **"CS"**
```
C = ConfigMap (Config, Clear text)
S = Secret (Sensitive, base64)
```

---

**Next: [09 - Scheduling & Resources](./09-scheduling-resources.md)** ➡️
