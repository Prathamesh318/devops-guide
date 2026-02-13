# 🐳 Docker Complete Learning Guide

> **"From Zero to Production — Master Containerization from the Ground Up"**

---

## 📚 Table of Contents

| # | Topic | File | Level |
|---|-------|------|-------|
| 1 | [Introduction & History](./01-introduction-history.md) | `01-introduction-history.md` | 🟢 Beginner |
| 2 | [Architecture & Internals](./02-architecture-internals.md) | `02-architecture-internals.md` | 🟢 Beginner |
| 3 | [Installation & Setup](./03-installation-setup.md) | `03-installation-setup.md` | 🟢 Beginner |
| 4 | [Images Deep Dive](./04-images-deep-dive.md) | `04-images-deep-dive.md` | 🟡 Intermediate |
| 5 | [Containers — Lifecycle & Management](./05-containers-lifecycle.md) | `05-containers-lifecycle.md` | 🟡 Intermediate |
| 6 | [Dockerfile Mastery](./06-dockerfile-mastery.md) | `06-dockerfile-mastery.md` | 🟡 Intermediate |
| 7 | [Networking Deep Dive](./07-networking-deep-dive.md) | `07-networking-deep-dive.md` | 🟡 Intermediate |
| 8 | [Storage — Volumes & Bind Mounts](./08-storage-volumes.md) | `08-storage-volumes.md` | 🟡 Intermediate |
| 9 | [Docker Compose](./09-docker-compose.md) | `09-docker-compose.md` | 🟡 Intermediate |
| 10 | [Docker Registry & Image Management](./10-registry-image-management.md) | `10-registry-image-management.md` | 🔴 Advanced |
| 11 | [Security Best Practices](./11-security-best-practices.md) | `11-security-best-practices.md` | 🔴 Advanced |
| 12 | [Production Best Practices & Optimization](./12-production-optimization.md) | `12-production-optimization.md` | 🔴 Advanced |
| 13 | [Hands-on Projects](./13-hands-on-projects.md) | `13-hands-on-projects.md` | 🟡 Practical |
| 14 | [Cheat Sheet & Quick Reference](./14-cheatsheet.md) | `14-cheatsheet.md` | 📋 Reference |

---

## 🎯 Quick Memory Shortcuts

### The Docker Acronym Family
```
┌─────────────────────────────────────────────────────────────┐
│  Docker   = Container platform (build, ship, run)            │
│  OCI      = Open Container Initiative (industry standard)    │
│  Compose  = Multi-container orchestration (YAML)             │
│  Swarm    = Docker's native clustering (mostly replaced by K8s) │
└─────────────────────────────────────────────────────────────┘
```

### Remember Architecture with **"CDRI"**
```
C = Client (docker CLI — sends commands)
D = Daemon (dockerd — does the actual work)
R = Registry (Docker Hub — stores images)
I = Images & Containers (blueprint vs running instance)
```

### Remember Storage with **"VBT"**
```
V = Volumes (Docker-managed, best for production)
B = Bind Mounts (host path mapped into container)
T = tmpfs (in-memory, temporary, Linux only)
```

### Remember Networking with **"BHO-N"**
```
B = Bridge (default, containers on same host)
H = Host (no isolation, use host's network)
O = Overlay (multi-host, Docker Swarm/K8s)
N = None (no network at all)
```

---

## 🗺️ Learning Path

```
┌─────────┐
│ Start   │
│ Here    │
└────┬────┘
     ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ 1. Introduction │───▶│ 2. Architecture │───▶│ 3. Installation │
└─────────────────┘    └─────────────────┘    └────────┬────────┘
                                                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ 6. Dockerfile   │◀───│ 5. Containers   │◀───│ 4. Images       │
└────────┬────────┘    └─────────────────┘    └─────────────────┘
         ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ 7. Networking   │───▶│ 8. Storage      │───▶│ 9. Compose      │
└─────────────────┘    └─────────────────┘    └────────┬────────┘
                                                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ 12. Production  │◀───│ 11. Security    │◀───│ 10. Registry    │
└────────┬────────┘    └─────────────────┘    └─────────────────┘
         ▼
┌─────────────────┐    ┌─────────────────┐
│ 13. Projects    │───▶│ 14. Cheat Sheet │───▶ 🎉 Docker Pro!
└─────────────────┘    └─────────────────┘
```

---

## 💡 Pro Tips Before You Start

1. **Containers are NOT VMs** — understand the difference before anything else
2. **Practice every command** — use your local machine, break things, experiment
3. **Build your own images** — don't just pull pre-built ones
4. **Read Dockerfiles of popular projects** — learn from nginx, postgres, node official images
5. **Use `docker system prune` regularly** — Docker eats disk space fast

---

## 🛠️ Prerequisites

Before diving in, ensure you have:
- ✅ Basic Linux/Unix commands (ls, cd, cat, grep)
- ✅ Understanding of processes and networking basics
- ✅ YAML syntax knowledge
- ✅ A machine with 4GB+ RAM
- ✅ Admin/root access to install Docker

---

**Ready to begin? Start with [01 - Introduction & History](./01-introduction-history.md)** 🚀
