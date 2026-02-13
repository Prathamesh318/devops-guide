# 🔐 Chapter 10: Security & RBAC

> **"Secure your cluster with proper access controls"**

---

## 🎯 K8s Security Overview

```
┌────────────────────────────────────────────────────────────────┐
│                K8s SECURITY LAYERS                              │
│                                                                 │
│   1. Authentication (WHO are you?)                             │
│      └─► Certificates, Tokens, OIDC                            │
│                                                                 │
│   2. Authorization (WHAT can you do?)                          │
│      └─► RBAC, ABAC, Webhook                                   │
│                                                                 │
│   3. Admission Control (IS this allowed?)                      │
│      └─► Pod Security, Resource Quotas                         │
│                                                                 │
│   Request ──► Auth ──► Authz ──► Admission ──► API             │
└────────────────────────────────────────────────────────────────┘
```

---

## 👤 Service Accounts

> **Identity for pods to interact with K8s API**

```yaml
# Create ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
---
# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: app
    image: myapp
```

### ServiceAccount Commands

```bash
kubectl get serviceaccounts
kubectl get sa  # short
kubectl create sa my-sa
kubectl describe sa my-sa
```

---

## 🎭 RBAC (Role-Based Access Control)

> **Define WHO can do WHAT on WHICH resources**

```
┌────────────────────────────────────────────────────────────────┐
│                    RBAC COMPONENTS                              │
│                                                                 │
│   ┌──────────────┐                    ┌──────────────┐         │
│   │    Role /    │ ◄─── defines ──── │    Rules     │         │
│   │ ClusterRole  │                    │ (permissions)│         │
│   └──────┬───────┘                    └──────────────┘         │
│          │                                                      │
│          │ binds to                                             │
│          ▼                                                      │
│   ┌──────────────┐                    ┌──────────────┐         │
│   │ RoleBinding/ │ ◄─── grants to ── │   Subject    │         │
│   │ClusterBinding│                    │ (User/SA)    │         │
│   └──────────────┘                    └──────────────┘         │
└────────────────────────────────────────────────────────────────┘
```

### Role Types

| Type | Scope |
|------|-------|
| **Role** | Single namespace |
| **ClusterRole** | Cluster-wide |
| **RoleBinding** | Binds Role in namespace |
| **ClusterRoleBinding** | Binds ClusterRole cluster-wide |

---

### Role Example (Namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### ClusterRole Example (Cluster-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

---

### RoleBinding Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: default
# Or for a user:
# - kind: User
#   name: jane
#   apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-binding
subjects:
- kind: User
  name: admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

---

## 📋 Common Verbs

| Verb | Description |
|------|-------------|
| `get` | Read single resource |
| `list` | List resources |
| `watch` | Watch for changes |
| `create` | Create resources |
| `update` | Modify resources |
| `patch` | Partial update |
| `delete` | Delete resources |
| `*` | All verbs |

---

## 🛠️ RBAC Commands

```bash
# List roles and bindings
kubectl get roles
kubectl get rolebindings
kubectl get clusterroles
kubectl get clusterrolebindings

# Create role imperatively
kubectl create role pod-reader --verb=get,list,watch --resource=pods

# Create binding
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=default:my-app-sa

# Check permissions
kubectl auth can-i list pods
kubectl auth can-i create deployments --as=jane
kubectl auth can-i '*' '*' --as=system:serviceaccount:default:my-sa
```

---

## 🛡️ Network Policies

> **Control pod-to-pod network traffic**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

---

## 🧠 Memory Shortcuts

### RBAC Components: **"RRBB"**
```
R = Role (namespace permissions)
R = (Cluster)Role (cluster permissions)
B = (Role)Binding (grants role in namespace)
B = (Cluster)RoleBinding (grants cluster-wide)
```

### Permission Formula
```
WHO (Subject) + WHAT (Role) + WHERE (Binding) = ACCESS
```

---

**Next: [11 - Advanced Concepts](./11-advanced-concepts.md)** ➡️
