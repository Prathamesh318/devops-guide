# 💾 Chapter 8: Storage — Volumes & Bind Mounts

> **"Containers are ephemeral — they come and go. Your DATA should not."**

---

## 🎯 The Storage Problem

```
┌──────────────────────────────────────────────────────────────────┐
│              WHY DO WE NEED DOCKER STORAGE?                       │
│                                                                   │
│  Problem: Container's writable layer is TEMPORARY                │
│                                                                   │
│  1. Container deleted → ALL data inside is GONE                  │
│  2. Data can't be easily shared between containers               │
│  3. Writing to container layer is SLOWER (copy-on-write)         │
│  4. Container layer mixes app code with runtime data             │
│                                                                   │
│  docker run -d --name db postgres                                │
│  # Insert 1 million rows over months...                          │
│  docker rm db                                                    │
│  # ALL DATA GONE! 💀                                             │
│                                                                   │
│  Solution: Store data OUTSIDE the container layer                │
│  → Volumes, Bind Mounts, or tmpfs                                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📊 Three Types of Storage

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│  ┌─── Container ───┐      ┌─── Host Filesystem ───────────────┐│
│  │                  │      │                                    ││
│  │   /app           │      │                                    ││
│  │   /var/lib/data ─┼──────┼─► Volume (/var/lib/docker/volumes)││
│  │                  │      │                                    ││
│  │   /config ───────┼──────┼─► Bind Mount (/home/user/config)  ││
│  │                  │      │                                    ││
│  │   /tmp/cache ────┼──────┼─► tmpfs (RAM only, no disk)       ││
│  │                  │      │                                    ││
│  └──────────────────┘      └────────────────────────────────────┘│
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Comparison

| Feature | Volume | Bind Mount | tmpfs |
|---------|--------|------------|-------|
| **Storage location** | Docker-managed (`/var/lib/docker/volumes/`) | Anywhere on host | RAM only |
| **Created by** | Docker | User (must exist) | Docker |
| **Managed by** | Docker CLI | You (manually) | Docker |
| **Shareable** | Yes (between containers) | Yes | No |
| **Survives container deletion** | ✅ Yes | ✅ Yes | ❌ No |
| **Performance** | Fast (native) | Fast (native) | Fastest (RAM) |
| **Backup** | `docker volume` commands | Standard file tools | N/A |
| **Use case** | Production data, databases | Dev configs, source code | Secrets, temp cache |
| **Works on all OS** | ✅ | ⚠️ Path differences | Linux only |

---

## 📦 Volumes (Production Standard)

### Why Volumes Are Best for Production

```
┌──────────────────────────────────────────────────────────────────┐
│  VOLUMES — Docker's recommended storage mechanism                │
│                                                                   │
│  ✅ Docker manages the directory                                 │
│  ✅ Works on Linux, Mac, Windows                                 │
│  ✅ Can be backed up with Docker commands                        │
│  ✅ Can use volume drivers (cloud storage, NFS, etc.)            │
│  ✅ Safer than bind mounts (no accidental host file access)      │
│  ✅ Can be pre-populated from container                          │
└──────────────────────────────────────────────────────────────────┘
```

### Volume Commands

```bash
# Create a volume
docker volume create my-data

# List volumes
docker volume ls

# Inspect volume details
docker volume inspect my-data
# Shows: Mountpoint, Driver, Labels, etc.

# Remove a volume
docker volume rm my-data

# Remove ALL unused volumes (⚠️ careful!)
docker volume prune
```

### Using Volumes with Containers

```bash
# Named volume (RECOMMENDED)
docker run -d --name postgres \
    -v pgdata:/var/lib/postgresql/data \
    postgres:15

# Same thing with --mount (more explicit, recommended for clarity)
docker run -d --name postgres \
    --mount source=pgdata,target=/var/lib/postgresql/data \
    postgres:15

# If volume doesn't exist, Docker creates it automatically

# Anonymous volume (Docker generates random name)
docker run -d -v /var/lib/postgresql/data postgres:15
# Volume name: random hash like abc123def456

# Read-only volume
docker run -d \
    -v config-vol:/app/config:ro \
    myapp
```

### Real-World Volume Examples

```bash
# PostgreSQL — data survives container recreation
docker volume create pg-data
docker run -d --name db \
    -v pg-data:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret \
    postgres:15

# Stop, remove, recreate — DATA IS STILL THERE!
docker rm -f db
docker run -d --name db \
    -v pg-data:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret \
    postgres:15
# All tables, rows, everything is preserved ✅

# Redis with persistent data
docker run -d --name redis \
    -v redis-data:/data \
    redis:alpine redis-server --save 60 1 --loglevel warning

# Sharing volume between containers
docker run -d --name writer -v shared-data:/data alpine sh -c "echo hello > /data/file.txt && sleep 3600"
docker run --rm -v shared-data:/data alpine cat /data/file.txt
# Output: hello
```

---

## 📁 Bind Mounts (Development Essential)

```
┌──────────────────────────────────────────────────────────────────┐
│  BIND MOUNTS — Map host directory into container                 │
│                                                                   │
│  ✅ Perfect for development (live code reload)                   │
│  ✅ Full control over host path                                  │
│  ❌ Host path must exist                                        │
│  ❌ Different paths on different OS (portability issue)          │
│  ❌ Container can modify host files (security risk)              │
└──────────────────────────────────────────────────────────────────┘
```

```bash
# Bind mount with -v (short syntax)
docker run -d -p 3000:3000 \
    -v $(pwd)/src:/app/src \
    myapp

# Bind mount with --mount (explicit syntax — preferred)
docker run -d -p 3000:3000 \
    --mount type=bind,source=$(pwd)/src,target=/app/src \
    myapp

# Read-only bind mount
docker run -d \
    -v $(pwd)/config:/app/config:ro \
    myapp

# Windows path example
docker run -d -v C:\Users\dev\project:/app myapp
```

### Development Workflow with Bind Mounts

```bash
# Live reload development setup
# Your code changes are INSTANTLY reflected in the container!

# Node.js with nodemon
docker run -d -p 3000:3000 \
    -v $(pwd):/app \
    -v /app/node_modules \
    myapp-dev

# The second -v is important:
# -v $(pwd):/app           → Mount your code
# -v /app/node_modules     → Anonymous volume for node_modules
#                             (prevents host's node_modules from
#                              overriding container's)

# Python with Flask auto-reload
docker run -d -p 5000:5000 \
    -v $(pwd):/app \
    -e FLASK_ENV=development \
    myapp-dev
```

---

## 🧊 tmpfs Mounts (Temporary In-Memory)

```bash
# Store sensitive data or cache in RAM (never touches disk)
docker run -d --name secure \
    --tmpfs /tmp:rw,size=100m \
    myapp

# Using --mount syntax
docker run -d --name secure \
    --mount type=tmpfs,target=/tmp,tmpfs-size=100m \
    myapp

# Use cases:
# • Storing secrets that should never touch disk
# • High-speed temporary caches
# • Test environments that need fast I/O
```

---

## 🔧 Volume Drivers (Advanced)

```bash
# Local driver (default)
docker volume create --driver local my-vol

# NFS mount
docker volume create --driver local \
    --opt type=nfs \
    --opt o=addr=192.168.1.100,rw \
    --opt device=:/path/to/share \
    nfs-data

# Use Docker volume plugins for cloud storage:
# • REX-Ray: AWS EBS, Azure Disk, GCE PD
# • Portworx: Cloud-native storage
# • NetApp Trident: Enterprise storage
```

---

## 💾 Backup and Restore Volumes

```bash
# BACKUP a volume to a tar file
docker run --rm \
    -v pgdata:/source:ro \
    -v $(pwd):/backup \
    alpine tar czf /backup/pgdata-backup.tar.gz -C /source .

# RESTORE from backup
docker run --rm \
    -v pgdata:/target \
    -v $(pwd):/backup \
    alpine tar xzf /backup/pgdata-backup.tar.gz -C /target

# How it works:
# 1. Create a temporary container
# 2. Mount the volume AND a host directory
# 3. Use tar to compress volume contents
# 4. Remove the temporary container (--rm)
```

---

## ⚠️ Common Storage Mistakes

```
┌──────────────────────────────────────────────────────────────────┐
│              COMMON MISTAKES                                      │
│                                                                   │
│  ❌ Mistake 1: Not using volumes for databases                   │
│     docker run postgres     ← data LOST when container dies!    │
│     ✅ docker run -v pgdata:/var/lib/postgresql/data postgres    │
│                                                                   │
│  ❌ Mistake 2: Using bind mounts in production                   │
│     Bind mounts are for DEV. Use named volumes in PROD.          │
│                                                                   │
│  ❌ Mistake 3: Forgetting to backup volumes                      │
│     Volumes are NOT backed up automatically!                     │
│     Set up regular backup jobs.                                  │
│                                                                   │
│  ❌ Mistake 4: Orphaned volumes eating disk space                │
│     Old volumes pile up. Run: docker volume prune                │
│     Check disk: docker system df                                 │
│                                                                   │
│  ❌ Mistake 5: Writing logs to container layer                   │
│     Logs in container layer = container grows huge               │
│     ✅ Use Docker log driver or volume-mounted log directory     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Memory Shortcuts for This Chapter

### Storage types: **"VBT"**
```
V = Volumes (Docker-managed, for production data)
B = Bind mounts (host directory, for development)
T = tmpfs (RAM only, for temp/sensitive data)
```

### When to use what:
```
Database → Named Volume (always!)
Dev code → Bind Mount (live reload)
Secrets  → tmpfs (never on disk)
Config   → Bind Mount (read-only) or ConfigMap in K8s
Logs     → Volume or log driver
```

---

## ❓ Quick Quiz

1. What happens to data in a container's writable layer when the container is deleted?
2. What is the difference between a volume and a bind mount?
3. Why should you use named volumes instead of anonymous volumes?
4. How do you backup a Docker volume?
5. When would you use a tmpfs mount?

<details>
<summary>Click for Answers</summary>

1. It's permanently deleted. The writable layer exists only for the lifetime of the container.
2. Volume = Docker-managed, stored in `/var/lib/docker/volumes/`, portable. Bind mount = you specify exact host path, host-dependent.
3. Named volumes are easier to identify, backup, and reference. Anonymous volumes get random hash names and are easy to lose track of.
4. Create a temporary container, mount the volume and a host directory, use tar to compress the volume contents to the host directory.
5. When you need fast temporary storage that should never be written to disk (secrets, caches, temp files).

</details>

---

**← Previous: [07 - Networking Deep Dive](./07-networking-deep-dive.md)** | **Next: [09 - Docker Compose](./09-docker-compose.md)** ➡️
