# 🏗️ Chapter 2: Docker Architecture & Internals

> **"Docker is NOT one single program — it's a client-server system with multiple components working together."**

---

## 🎯 Docker Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                     DOCKER ARCHITECTURE                               │
│                                                                       │
│  ┌──────────────┐         REST API          ┌──────────────────────┐│
│  │ Docker CLI   │ ─────────────────────────► │    Docker Daemon     ││
│  │ (docker)     │         /var/run/          │    (dockerd)         ││
│  │              │      docker.sock           │                      ││
│  │ "docker run" │                            │  ┌────────────────┐ ││
│  │ "docker build"│                           │  │   containerd   │ ││
│  │ "docker pull" │                           │  │                │ ││
│  └──────────────┘                            │  │  ┌──────────┐ │ ││
│     CLIENT                                   │  │  │  runc    │ │ ││
│                                              │  │  │ (creates │ │ ││
│  ┌──────────────┐                            │  │  │ container│ │ ││
│  │ Docker       │                            │  │  └──────────┘ │ ││
│  │ Desktop      │                            │  └────────────────┘ ││
│  │ (GUI)        │                            │                      ││
│  └──────────────┘                            │  Manages:            ││
│     CLIENT                                   │  • Images             ││
│                                              │  • Containers         ││
│                                              │  • Networks           ││
│                                              │  • Volumes            ││
│                                              └──────────────────────┘│
│                                                    SERVER             │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────────┐│
│  │                     DOCKER REGISTRY                               ││
│  │  Docker Hub │ AWS ECR │ GCR │ GitHub GHCR │ Private Registry     ││
│  └──────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────┘
```

---

## 🧩 Component Deep Dive

### 1. Docker Client (CLI)

```
┌──────────────────────────────────────────────────────────────────┐
│  DOCKER CLIENT (docker)                                          │
│                                                                   │
│  What: The command-line tool you interact with                   │
│  Where: Your terminal                                             │
│  How: Sends REST API calls to Docker Daemon                      │
│                                                                   │
│  When you type:  docker run nginx                                │
│                    │                                              │
│                    ▼                                              │
│  Client sends:   POST /containers/create                         │
│                  POST /containers/{id}/start                     │
│                    │                                              │
│                    ▼                                              │
│  To:             Docker Daemon (via /var/run/docker.sock)        │
│                                                                   │
│  Key Point: Client and Daemon CAN run on different machines     │
│  Example:   docker -H tcp://remote-server:2375 ps               │
└──────────────────────────────────────────────────────────────────┘
```

### 2. Docker Daemon (dockerd)

```
┌──────────────────────────────────────────────────────────────────┐
│  DOCKER DAEMON (dockerd)                                         │
│                                                                   │
│  What: Background service that does ALL the heavy lifting        │
│  Where: Runs as a system service (systemd)                       │
│                                                                   │
│  Responsibilities:                                                │
│  ├── Listens for API requests from Docker Client                │
│  ├── Manages Docker objects (images, containers, networks)       │
│  ├── Communicates with other daemons (Swarm mode)               │
│  ├── Delegates container creation to containerd                  │
│  └── Handles image building (build context)                     │
│                                                                   │
│  Config file: /etc/docker/daemon.json                            │
│  {                                                                │
│    "storage-driver": "overlay2",                                 │
│    "log-driver": "json-file",                                    │
│    "log-opts": { "max-size": "10m", "max-file": "3" },          │
│    "default-address-pools": [{"base":"172.80.0.0/16","size":24}]│
│  }                                                                │
└──────────────────────────────────────────────────────────────────┘
```

### 3. containerd

```
┌──────────────────────────────────────────────────────────────────┐
│  containerd                                                       │
│                                                                   │
│  What: Industry-standard container runtime                       │
│  Why:  Docker donated it to CNCF (cloud-native foundation)       │
│                                                                   │
│  Responsibilities:                                                │
│  ├── Manages complete container lifecycle                        │
│  │   (create → start → stop → delete)                           │
│  ├── Pulls and stores images                                    │
│  ├── Manages storage (snapshots)                                │
│  ├── Manages networking (CNI)                                   │
│  └── Calls runc to actually create containers                   │
│                                                                   │
│  Key insight:                                                     │
│  Kubernetes also uses containerd directly (without dockerd!)     │
│                                                                   │
│  Docker:      CLI → dockerd → containerd → runc → container     │
│  Kubernetes:  kubelet → containerd → runc → container            │
└──────────────────────────────────────────────────────────────────┘
```

### 4. runc

```
┌──────────────────────────────────────────────────────────────────┐
│  runc                                                             │
│                                                                   │
│  What: Low-level container runtime                               │
│  What it does: Creates the actual container process              │
│                                                                   │
│  It configures:                                                   │
│  ├── Linux namespaces (PID, NET, MNT, UTS, IPC, User)          │
│  ├── Cgroups (CPU, memory, I/O limits)                          │
│  ├── Security (seccomp, AppArmor, SELinux)                      │
│  ├── Root filesystem (from image layers)                         │
│  └── Starts the container's main process (PID 1 inside)         │
│                                                                   │
│  Once container is running, runc EXITS.                          │
│  containerd manages the container from there.                    │
│                                                                   │
│  Think of runc as: "The construction worker who builds the      │
│  room, then leaves. containerd is the building manager."         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔄 What Happens When You Run `docker run nginx`?

```
Step-by-step:

1. You type: docker run nginx
   │
   ▼
2. Docker CLI sends API request to Docker Daemon
   │  POST /containers/create  { "Image": "nginx" }
   │
   ▼
3. Docker Daemon checks: "Do I have the nginx image locally?"
   │
   ├── YES → Skip to step 5
   │
   └── NO → Step 4: Pull from registry
       │
       ▼
4. Daemon pulls image from Docker Hub
   │  - Downloads each layer
   │  - Stores in /var/lib/docker/overlay2/
   │
   ▼
5. Daemon tells containerd: "Create a container from this image"
   │
   ▼
6. containerd prepares the container bundle:
   │  - Combines image layers into a single filesystem view
   │  - Creates OCI runtime specification (config.json)
   │
   ▼
7. containerd calls runc to create the container:
   │  - runc creates namespaces (PID, NET, MNT...)
   │  - runc sets up cgroups (resource limits)
   │  - runc starts the process (nginx master process)
   │  - runc exits
   │
   ▼
8. Container is running! ✅
   │  - containerd monitors the container process
   │  - Docker Daemon reports status back to CLI
   │  - You see: container ID printed in terminal
```

---

## 📁 Docker Storage Architecture

### Image Layers

```
┌──────────────────────────────────────────────────────────────────┐
│           HOW IMAGE LAYERS WORK                                    │
│                                                                   │
│   Dockerfile:                   Image Layers:                    │
│                                                                   │
│   FROM ubuntu:22.04     ──►    Layer 1: Ubuntu base (77MB)       │
│   RUN apt install python ──►   Layer 2: Python installed (35MB)  │
│   COPY app.py /app/     ──►    Layer 3: App code (2KB)           │
│   CMD ["python", "app.py"]     (metadata only, no layer)         │
│                                                                   │
│   ┌────────────────────────────────────────────────┐             │
│   │ Layer 3: COPY app.py          (2KB)    READ-ONLY │           │
│   ├────────────────────────────────────────────────┤             │
│   │ Layer 2: RUN apt install      (35MB)   READ-ONLY │           │
│   ├────────────────────────────────────────────────┤             │
│   │ Layer 1: FROM ubuntu          (77MB)   READ-ONLY │           │
│   └────────────────────────────────────────────────┘             │
│                                                                   │
│   When you RUN a container, Docker adds:                         │
│                                                                   │
│   ┌────────────────────────────────────────────────┐             │
│   │ Container Layer (thin, writable)  READ-WRITE   │ ← New!     │
│   ├────────────────────────────────────────────────┤             │
│   │ Layer 3: COPY app.py          (2KB)    READ-ONLY │           │
│   ├────────────────────────────────────────────────┤             │
│   │ Layer 2: RUN apt install      (35MB)   READ-ONLY │           │
│   ├────────────────────────────────────────────────┤             │
│   │ Layer 1: FROM ubuntu          (77MB)   READ-ONLY │           │
│   └────────────────────────────────────────────────┘             │
│                                                                   │
│   KEY: All containers from same image SHARE the read-only layers │
│   Only the writable layer is unique per container.               │
│   This is why 100 nginx containers don't use 100× storage!      │
└──────────────────────────────────────────────────────────────────┘
```

### Copy-on-Write (CoW)

```
When a container modifies a file from a read-only layer:

1. Docker COPIES the file to the writable container layer
2. Modifications happen on the COPY
3. Original layer remains unchanged

Example:
  Container wants to edit /etc/nginx/nginx.conf (from Layer 2)
  
  Before edit:
  ┌──────────────────────────────┐
  │ Container Layer (empty)      │ ← writable
  ├──────────────────────────────┤
  │ Layer 2: nginx.conf (orig)  │ ← read-only
  └──────────────────────────────┘
  
  After edit:
  ┌──────────────────────────────┐
  │ Container Layer              │ ← writable
  │   └─ nginx.conf (modified)  │ ← this version is used
  ├──────────────────────────────┤
  │ Layer 2: nginx.conf (orig)  │ ← still unchanged
  └──────────────────────────────┘
```

---

## 🌐 Docker Socket — The Power Connector

```
┌──────────────────────────────────────────────────────────────────┐
│  DOCKER SOCKET (/var/run/docker.sock)                            │
│                                                                   │
│  What: Unix socket for communication between CLI and Daemon      │
│                                                                   │
│  CLI ──── docker.sock ────► Daemon                               │
│                                                                   │
│  ⚠️  SECURITY WARNING:                                           │
│  Anyone with access to docker.sock has ROOT-equivalent access!   │
│                                                                   │
│  Why? Because Docker can:                                         │
│  • Mount host filesystem into containers                         │
│  • Run containers as privileged (full host access)               │
│  • Access host network                                           │
│                                                                   │
│  Rule: NEVER expose docker.sock to the internet                  │
│  Rule: NEVER mount docker.sock into containers (unless required) │
│                                                                   │
│  TCP socket (for remote access):                                  │
│  docker -H tcp://remote-host:2375 ps   (NO TLS — INSECURE!)     │
│  docker -H tcp://remote-host:2376 ps   (with TLS — SECURE)      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Docker Objects

### The Complete Object Model

```
┌──────────────────────────────────────────────────────────────────┐
│                    DOCKER OBJECTS                                  │
│                                                                   │
│  IMAGES ──────────────► CONTAINERS                               │
│  (templates)           (running instances)                        │
│    │                                                              │
│    │  Built from                                                 │
│    │                                                              │
│  DOCKERFILES                                                     │
│  (build instructions)                                            │
│                                                                   │
│  NETWORKS ─────── Connect containers together                    │
│  │  bridge        (default, same host)                           │
│  │  host          (no isolation, host network)                   │
│  │  overlay       (multi-host, Swarm)                            │
│  │  none          (no network)                                   │
│  │  macvlan       (physical network level)                       │
│                                                                   │
│  VOLUMES ──────── Persistent storage                             │
│  │  named volumes    (Docker managed)                            │
│  │  bind mounts      (host directory)                            │
│  │  tmpfs            (in-memory)                                 │
│                                                                   │
│  Everything is managed via:                                      │
│  docker image ...                                                │
│  docker container ...                                            │
│  docker network ...                                              │
│  docker volume ...                                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📊 How Docker Stores Data on Disk

```
/var/lib/docker/              ← Docker's home directory
├── overlay2/                 ← Image and container layers (default driver)
│   ├── abc123.../            ← Layer 1 data
│   ├── def456.../            ← Layer 2 data
│   └── ...
├── containers/               ← Container metadata and logs
│   ├── <container-id>/
│   │   ├── config.v2.json   ← Container configuration
│   │   ├── hostname          ← Container hostname
│   │   ├── resolv.conf       ← DNS configuration
│   │   └── <container-id>-json.log  ← Container logs!
│   └── ...
├── image/                    ← Image metadata
├── network/                  ← Network configurations
├── volumes/                  ← Named volumes
│   ├── my-vol/
│   │   └── _data/           ← Actual volume data
│   └── ...
├── tmp/                      ← Build cache and temp files
└── buildkit/                 ← BuildKit cache

⚠️  This grows over time → use: docker system prune
```

---

## 🔌 Docker vs Podman vs containerd

```
┌──────────────────────────────────────────────────────────────────┐
│                CONTAINER RUNTIMES COMPARISON                      │
│                                                                   │
│  Docker (dockerd + containerd + runc)                            │
│  ├── Full platform: build + run + push + compose                │
│  ├── Client-server architecture (daemon always running)          │
│  ├── Requires root (or rootless mode)                           │
│  └── Most popular, best tooling and docs                        │
│                                                                   │
│  Podman (no daemon!)                                             │
│  ├── Drop-in replacement for Docker CLI                         │
│  ├── Daemonless: each command runs as a process                 │
│  ├── Rootless by default (better security)                      │
│  ├── Compatible: `alias docker=podman`                          │
│  └── Preferred by Red Hat / Fedora / CentOS                     │
│                                                                   │
│  containerd (standalone)                                         │
│  ├── CNCF graduated project                                     │
│  ├── Used by Kubernetes directly (no Docker needed!)             │
│  ├── No build capability (needs BuildKit separately)             │
│  └── Lower-level, not meant for direct developer use            │
│                                                                   │
│  For learning and daily use → Docker                            │
│  For enterprise/security     → Podman                           │
│  For Kubernetes              → containerd                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Memory Shortcuts for This Chapter

### Architecture: **"CDRC"** (Think: CD-RC)
```
C = Client (CLI sends commands)
D = Daemon (dockerd does the work)
R = Runtime (containerd + runc creates containers)
C = (not related to above) containerd manages lifecycle
```

### Image Layers: **"BSRW"**
```
B = Base layer (FROM — OS image)
S = Software layers (RUN — install stuff)
R = Read-only (all image layers)
W = Writable (container layer — only one)
```

### What happens on `docker run`: **"PCCS"**
```
P = Pull image (if not local)
C = Create container (containerd)
C = Configure (namespaces + cgroups via runc)
S = Start process (PID 1 in container)
```

---

## ❓ Quick Quiz

1. What is the difference between dockerd and containerd?
2. What is the Docker socket and why is it a security risk?
3. How do image layers work? Why is this efficient?
4. What is Copy-on-Write?
5. What happens step-by-step when you run `docker run nginx`?

<details>
<summary>Click for Answers</summary>

1. dockerd is the Docker daemon that manages the overall Docker system; containerd is the lower-level container runtime that manages the container lifecycle. dockerd delegates to containerd.
2. `/var/run/docker.sock` is the Unix socket for CLI-daemon communication. It's a security risk because access to it gives root-equivalent privileges on the host.
3. Each Dockerfile instruction creates a layer. Layers are stacked and shared between images/containers. 100 containers from the same image share all read-only layers.
4. When a container modifies a file from a read-only layer, Docker copies the file to the writable layer first, then modifies the copy. Original layer stays unchanged.
5. CLI → API to daemon → check local image → pull if needed → containerd creates bundle → runc creates namespaces/cgroups → starts process → container running

</details>

---

**← Previous: [01 - Introduction & History](./01-introduction-history.md)** | **Next: [03 - Installation & Setup](./03-installation-setup.md)** ➡️
