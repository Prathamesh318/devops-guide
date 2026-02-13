# 💾 Chapter 6: Storage in Kubernetes

> **"Containers are ephemeral, but data must persist!"**

---

## 🎯 Storage Concepts Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                    K8s STORAGE HIERARCHY                          │
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │                   StorageClass                           │    │
│   │         (Defines HOW storage is provisioned)            │    │
│   └───────────────────────┬─────────────────────────────────┘    │
│                           │ creates                              │
│                           ▼                                       │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │               PersistentVolume (PV)                      │    │
│   │          (Actual storage in the cluster)                │    │
│   └───────────────────────┬─────────────────────────────────┘    │
│                           │ binds to                             │
│                           ▼                                       │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │            PersistentVolumeClaim (PVC)                   │    │
│   │              (Pod's request for storage)                │    │
│   └───────────────────────┬─────────────────────────────────┘    │
│                           │ used by                              │
│                           ▼                                       │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │                       Pod                                │    │
│   │                  (Uses the storage)                      │    │
│   └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

**Memory: "SCP" (like the copy command!)** - StorageClass → PV → PVC → Pod

---

## 📁 Volume Types

### Ephemeral Volumes (Die with Pod)

| Type | Description |
|------|-------------|
| **emptyDir** | Empty directory, dies with pod |
| **configMap** | Mount ConfigMap as files |
| **secret** | Mount Secrets as files |

### Persistent Volumes (Survive Pod)

| Type | Description |
|------|-------------|
| **hostPath** | Mount from node filesystem |
| **nfs** | Network File System |
| **awsElasticBlockStore** | AWS EBS |
| **azureDisk** | Azure Disk |
| **gcePersistentDisk** | GCP Persistent Disk |

---

## 📦 emptyDir Volume

> **Temporary storage that lives as long as the pod**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo hello > /data/message; sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox
    command: ["sh", "-c", "cat /data/message; sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
```

**Use cases:** Scratch space, sharing between containers in a pod

---

## 💽 Persistent Volume (PV)

> **Admin-provisioned storage resource in the cluster**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

### Access Modes

| Mode | Abbreviation | Description |
|------|--------------|-------------|
| **ReadWriteOnce** | RWO | Read/write by single node |
| **ReadOnlyMany** | ROX | Read-only by many nodes |
| **ReadWriteMany** | RWX | Read/write by many nodes |

### Reclaim Policies

| Policy | What Happens |
|--------|-------------|
| **Retain** | Keep data, manual cleanup |
| **Delete** | Delete storage when PVC deleted |
| **Recycle** | Basic scrub (`rm -rf`) - deprecated |

### PV Commands

```bash
kubectl get pv
kubectl describe pv pv-database
```

---

## 📋 Persistent Volume Claim (PVC)

> **Pod's request for storage - like a ticket to claim a PV**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

### Using PVC in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: password123
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: pvc-database
```

### PVC Commands

```bash
kubectl get pvc
kubectl describe pvc pvc-database
```

---

## 🏭 StorageClass

> **Enables dynamic provisioning - auto-creates PVs**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### Dynamic Provisioning Flow

```
┌────────────────────────────────────────────────────────────┐
│             DYNAMIC PROVISIONING                            │
│                                                             │
│   1. PVC created with storageClassName: fast-storage       │
│                      │                                      │
│                      ▼                                      │
│   2. StorageClass triggers provisioner                     │
│                      │                                      │
│                      ▼                                      │
│   3. Cloud provider creates actual disk (EBS, etc.)        │
│                      │                                      │
│                      ▼                                      │
│   4. PV automatically created and bound to PVC             │
│                      │                                      │
│                      ▼                                      │
│   5. Pod uses PVC → Uses storage!                          │
└────────────────────────────────────────────────────────────┘
```

### Common StorageClass Provisioners

| Provider | Provisioner |
|----------|-------------|
| AWS EBS | `kubernetes.io/aws-ebs` |
| Azure Disk | `kubernetes.io/azure-disk` |
| GCP PD | `kubernetes.io/gce-pd` |
| NFS | External provisioners |

### StorageClass Commands

```bash
kubectl get storageclass
kubectl get sc  # short
kubectl describe sc fast-storage
```

---

## 📊 Complete Storage Example

```yaml
# 1. StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: k8s.io/minikube-hostpath
---
# 2. PVC (auto-creates PV with StorageClass)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
---
# 3. Pod using PVC
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-pvc
```

---

## 🧠 Memory Shortcuts

### Storage Components: **"SCP"**
```
S = StorageClass (how to provision)
C = Claim (PVC - what pod wants)
P = PersistentVolume (actual storage)
```

### Access Modes: **"RWO/ROX/RWX"**
```
RWO = One node writes
ROX = Many read only
RWX = Many read/write
```

---

**Next: [07 - Networking & Services](./07-networking-services.md)** ➡️
