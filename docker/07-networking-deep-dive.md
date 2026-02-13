# 🌐 Chapter 7: Docker Networking Deep Dive

> **"Containers need to talk to each other, and to the outside world. Docker networking makes this possible."**

---

## 🎯 Networking Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                 DOCKER NETWORK DRIVERS                            │
│                                                                   │
│  Driver     │ Use Case                    │ Scope               │
│  ───────────┼─────────────────────────────┼───────────────────── │
│  bridge     │ Default, containers on same │ Single host         │
│             │ host talk to each other     │                     │
│  host       │ No isolation, use host's    │ Single host         │
│             │ network directly            │                     │
│  none       │ No networking at all        │ Single host         │
│  overlay    │ Containers across multiple  │ Multi-host (Swarm)  │
│             │ hosts                       │                     │
│  macvlan    │ Assign real MAC address,    │ Single host         │
│             │ appear as physical device   │                     │
│  ipvlan     │ Similar to macvlan,         │ Single host         │
│             │ shares host MAC             │                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🌉 Bridge Network (Default)

```
┌──────────────────────────────────────────────────────────────────┐
│              BRIDGE NETWORK                                       │
│                                                                   │
│  HOST MACHINE                                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  ┌──────────┐    docker0 bridge    ┌──────────┐         │   │
│  │  │ Container │    (172.17.0.1)     │ Container │         │   │
│  │  │    A      │◄──────────────────►│    B      │         │   │
│  │  │172.17.0.2 │                     │172.17.0.3 │         │   │
│  │  └──────────┘                      └──────────┘         │   │
│  │       │                                 │                │   │
│  │       └──────────── veth ──────────────┘                │   │
│  │                      │                                   │   │
│  │              ┌───────┴───────┐                          │   │
│  │              │  docker0      │                          │   │
│  │              │  bridge       │                          │   │
│  │              └───────┬───────┘                          │   │
│  │                      │ NAT                              │   │
│  │              ┌───────┴───────┐                          │   │
│  │              │    eth0       │                          │   │
│  │              │  (host NIC)   │                          │   │
│  │              └───────────────┘                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Default bridge (docker0):                                       │
│  • Containers can communicate via IP                             │
│  • NO DNS resolution between containers                          │
│  • All containers on same bridge can see each other             │
│                                                                   │
│  User-defined bridge (ALWAYS use this): ✅                       │
│  • DNS resolution by container NAME                              │
│  • Better isolation                                              │
│  • Configurable                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Bridge Network Commands

```bash
# Default bridge (docker0) — containers connect here by default
docker run -d --name web nginx       # Connects to default bridge

# Create a user-defined bridge (RECOMMENDED)
docker network create my-network

# Run containers on the custom network
docker run -d --name web --network my-network nginx
docker run -d --name api --network my-network node:alpine

# Now 'api' can reach 'web' by name!
# From inside api container: curl http://web:80    ← DNS works!

# On default bridge, DNS does NOT work:
# curl http://web:80   → FAILS
# curl http://172.17.0.2:80  → Works (IP only)
```

### Why User-Defined Bridge is Better

| Feature | Default Bridge | User-Defined Bridge |
|---------|---------------|-------------------|
| DNS resolution | ❌ (IP only) | ✅ (by container name) |
| Isolation | All containers share it | Only chosen containers |
| Connect/disconnect live | ❌ Must recreate | ✅ `docker network connect` |
| Custom subnet | ❌ | ✅ |
| Use in production | ❌ Never | ✅ Always |

---

## 🏠 Host Network

```
┌──────────────────────────────────────────────────────────────────┐
│              HOST NETWORK                                         │
│                                                                   │
│  Container uses the HOST's network stack directly                │
│  NO network isolation, NO port mapping needed                    │
│                                                                   │
│  docker run --network host nginx                                │
│  → nginx listens on host's port 80 directly                    │
│  → No need for -p 80:80                                        │
│                                                                   │
│  ✅ Use when:                                                    │
│  • Maximum network performance needed                            │
│  • Container needs to see all host traffic                       │
│  • No port conflicts with host services                          │
│                                                                   │
│  ❌ Don't use when:                                              │
│  • Multiple containers need same port                            │
│  • You need network isolation                                    │
│  • Running on Docker Desktop (not supported on Mac/Windows)      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🚫 None Network

```bash
# Container with NO network at all
docker run --network none alpine
# No eth0, no connectivity, only loopback (127.0.0.1)

# Use for: Security-sensitive containers that should NEVER
# access the network (data processing, crypto, etc.)
```

---

## 🔧 Network Management Commands

```bash
# List networks
docker network ls

# Create network with options
docker network create \
    --driver bridge \
    --subnet 172.20.0.0/16 \
    --gateway 172.20.0.1 \
    --ip-range 172.20.240.0/20 \
    my-custom-network

# Inspect network (see connected containers)
docker network inspect my-network

# Connect running container to network
docker network connect my-network existing-container

# Disconnect container from network
docker network disconnect my-network existing-container

# Remove network
docker network rm my-network

# Remove all unused networks
docker network prune
```

---

## 🔗 Container Communication Patterns

### Pattern 1: Containers on Same Network (Recommended)

```bash
# Create dedicated network
docker network create app-net

# Start database
docker run -d --name postgres \
    --network app-net \
    -e POSTGRES_PASSWORD=secret \
    postgres:15-alpine

# Start application (connects to DB by name!)
docker run -d --name api \
    --network app-net \
    -e DATABASE_URL=postgresql://postgres:secret@postgres:5432/mydb \
    -p 3000:3000 \
    myapp:latest

# 'api' container connects to 'postgres' by name
# DNS resolves 'postgres' → 172.20.0.2 (container's IP)
```

### Pattern 2: Container-to-Host Communication

```bash
# From inside a container, reach the host machine:

# Linux:
# Use host.docker.internal (Docker Desktop) or
# Use docker0 bridge IP: 172.17.0.1

# Docker Desktop (Mac/Windows):
# host.docker.internal is automatically available
# docker run -e DB_HOST=host.docker.internal myapp

# Linux (add host entry manually):
# docker run --add-host=host.docker.internal:host-gateway myapp
```

### Pattern 3: Multi-Network Isolation

```bash
# Frontend network (web + api)
docker network create frontend

# Backend network (api + db)
docker network create backend

# Web server — only on frontend
docker run -d --name web --network frontend nginx

# API server — on BOTH networks (bridge between front and back)
docker run -d --name api --network frontend myapi
docker network connect backend api

# Database — only on backend
docker run -d --name db --network backend postgres

# Result:
# web ←→ api ✅ (both on frontend)
# api ←→ db  ✅ (both on backend)
# web ←→ db  ❌ (different networks, no route!)
```

```
┌──────────────────────────────────────────────────────────────────┐
│              MULTI-NETWORK ISOLATION                              │
│                                                                   │
│  ┌───────────────── frontend network ──────────────────┐        │
│  │  ┌──────────┐                      ┌──────────┐    │        │
│  │  │   web    │ ◄─── can talk ────► │   api    │    │        │
│  │  │  :80     │                      │  :3000   │    │        │
│  │  └──────────┘                      └──────────┘    │        │
│  └─────────────────────────────────────────┼──────────┘        │
│                                            │ api is on         │
│  ┌─────────────────────────────────────────┼──────────┐        │
│  │  ┌──────────┐                      ┌────┴─────┐    │        │
│  │  │   db     │ ◄─── can talk ────► │   api    │    │        │
│  │  │  :5432   │                      │  :3000   │    │        │
│  │  └──────────┘                      └──────────┘    │        │
│  └───────────────── backend network ───────────────────┘        │
│                                                                   │
│  web ←✖→ db    (ISOLATED — can't communicate!)                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔒 DNS Resolution in Docker

```
┌──────────────────────────────────────────────────────────────────┐
│              DOCKER DNS (Embedded DNS Server: 127.0.0.11)        │
│                                                                   │
│  User-defined networks have built-in DNS:                        │
│                                                                   │
│  Container name → IP address                                     │
│  "postgres"     → 172.20.0.2                                    │
│  "redis"        → 172.20.0.3                                    │
│  "api"          → 172.20.0.4                                    │
│                                                                   │
│  Network aliases (multiple names for same container):            │
│  docker run --network my-net --network-alias db postgres         │
│  Now: both "postgres" AND "db" resolve to the container          │
│                                                                   │
│  Custom DNS:                                                      │
│  docker run --dns 8.8.8.8 myapp   (custom DNS server)           │
│  docker run --dns-search example.com myapp                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📊 Port Mapping Deep Dive

```bash
# Formats
-p hostPort:containerPort         # Specific mapping
-p hostPort:containerPort/udp     # UDP protocol
-p 127.0.0.1:hostPort:containerPort  # Localhost only
-P                                # Auto-map all EXPOSE'd ports

# Examples
docker run -p 8080:80 nginx       # http://localhost:8080
docker run -p 80:80 -p 443:443 nginx  # Multiple ports
docker run -p 127.0.0.1:3306:3306 mysql  # Only local access

# See port mappings
docker port my-container
# 80/tcp -> 0.0.0.0:8080
# 443/tcp -> 0.0.0.0:8443
```

---

## 🧠 Memory Shortcuts for This Chapter

### Network types: **"BHO-N"**
```
B = Bridge (containers same host, use custom not default!)
H = Host (no isolation, max performance)
O = Overlay (multi-host, Swarm/K8s)
N = None (no network, security)
```

### Key rules:
```
1. ALWAYS use user-defined bridge (not default docker0)
2. DNS only works on user-defined networks
3. Use multi-network for isolation (front ←→ api ←→ db)
4. Never expose DB ports to host in production
```

---

## ❓ Quick Quiz

1. Why should you use a user-defined bridge instead of the default bridge?
2. How does DNS resolution work in Docker networking?
3. How would you isolate a database so only the API can reach it?
4. What is the host network mode and when would you use it?
5. How does a container communicate with a service on the host machine?

<details>
<summary>Click for Answers</summary>

1. User-defined bridge provides DNS resolution by container name, better isolation, and ability to connect/disconnect containers dynamically.
2. Docker runs an embedded DNS server (127.0.0.11) on user-defined networks. It resolves container names and network aliases to container IPs.
3. Create two networks (frontend, backend). Put web on frontend, DB on backend, and API on both. Web can't reach DB directly.
4. Host mode removes network isolation — container shares the host's network stack. Use when you need maximum network performance.
5. Use `host.docker.internal` (Docker Desktop) or the docker0 bridge gateway IP (172.17.0.1 on Linux).

</details>

---

**← Previous: [06 - Dockerfile Mastery](./06-dockerfile-mastery.md)** | **Next: [08 - Storage & Volumes](./08-storage-volumes.md)** ➡️
