# 📦 Chapter 5: Containers — Lifecycle & Management

> **"A container is just a process. Understanding its lifecycle is understanding Docker."**

---

## 🎯 Container Lifecycle

```
┌──────────────────────────────────────────────────────────────────┐
│                   CONTAINER LIFECYCLE                              │
│                                                                   │
│  Image                                                            │
│    │                                                              │
│    │  docker create                                               │
│    ▼                                                              │
│  CREATED ─────────────────────────────────────────────────────── │
│    │                                                              │
│    │  docker start                                                │
│    ▼                                                              │
│  RUNNING ◄──────────────────────────────────────────┐           │
│    │   │                                             │           │
│    │   │  docker pause                docker unpause │           │
│    │   ▼                                             │           │
│    │  PAUSED ────────────────────────────────────────┘           │
│    │                                                              │
│    │  docker stop (SIGTERM → 10s → SIGKILL)                      │
│    ▼                                                              │
│  STOPPED (EXITED)                                                │
│    │    │                                                         │
│    │    │  docker start (restart it)                              │
│    │    └──────────────────► RUNNING                             │
│    │                                                              │
│    │  docker rm                                                   │
│    ▼                                                              │
│  DELETED (gone forever)                                          │
│                                                                   │
│  Shortcut: docker run = docker create + docker start             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🚀 Running Containers

### Basic Run Commands

```bash
# Run a container (foreground — attached to terminal)
docker run nginx
# Ctrl+C to stop

# Run in background (detached mode — most common)
docker run -d nginx

# Run with a name (instead of random name)
docker run -d --name my-nginx nginx

# Run with port mapping
docker run -d -p 8080:80 --name web nginx
# Host port 8080 → Container port 80
# Access at: http://localhost:8080

# Run interactive (for shells)
docker run -it ubuntu bash
# -i = interactive (keep STDIN open)
# -t = allocate a pseudo-TTY (terminal)

# Run and auto-remove when stopped
docker run --rm nginx          # Container deleted when it exits

# Run with environment variables
docker run -d -e MYSQL_ROOT_PASSWORD=secret mysql:8

# Run with resource limits
docker run -d --memory=256m --cpus=0.5 nginx

# Run with restart policy
docker run -d --restart=always nginx
```

### Port Mapping Explained

```
┌──────────────────────────────────────────────────────────────────┐
│                   PORT MAPPING                                    │
│                                                                   │
│  docker run -p 8080:80 nginx                                    │
│                                                                   │
│  HOST                          CONTAINER                         │
│  ┌──────────────────┐         ┌──────────────────┐              │
│  │                  │         │                  │              │
│  │  Port 8080 ◄────────────────── Port 80       │              │
│  │                  │         │  (nginx listens) │              │
│  │                  │         │                  │              │
│  └──────────────────┘         └──────────────────┘              │
│                                                                   │
│  Multiple formats:                                                │
│  -p 8080:80           Host 8080 → Container 80                  │
│  -p 80:80             Same port                                   │
│  -p 127.0.0.1:8080:80 Only accessible from localhost            │
│  -p 8080:80/udp       UDP protocol                               │
│  -P (uppercase)       Auto-map to random host port              │
│                                                                   │
│  Multiple ports:                                                  │
│  -p 8080:80 -p 8443:443                                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📋 Managing Containers

### Listing Containers

```bash
# List running containers
docker ps
# OR
docker container ls

# List ALL containers (including stopped)
docker ps -a

# List only IDs
docker ps -q          # Running only
docker ps -aq         # All

# Format output
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Filter containers
docker ps --filter "status=exited"
docker ps --filter "name=web"
docker ps --filter "ancestor=nginx"
```

### Stopping & Starting

```bash
# Stop a running container (graceful — SIGTERM, then SIGKILL after 10s)
docker stop my-nginx

# Stop with custom timeout
docker stop -t 30 my-nginx    # Wait 30s before SIGKILL

# Kill immediately (SIGKILL — no graceful shutdown)
docker kill my-nginx

# Start a stopped container
docker start my-nginx

# Restart a container
docker restart my-nginx

# Pause a container (freeze all processes — SIGSTOP)
docker pause my-nginx

# Unpause
docker unpause my-nginx
```

### Removing Containers

```bash
# Remove a stopped container
docker rm my-nginx

# Force remove a running container
docker rm -f my-nginx

# Remove all stopped containers
docker container prune

# Remove ALL containers (running + stopped)
docker rm -f $(docker ps -aq)
```

---

## 🔍 Inspecting & Debugging Containers

### Viewing Logs

```bash
# View container logs
docker logs my-nginx

# Follow logs in real-time (like tail -f)
docker logs -f my-nginx

# Show last 100 lines
docker logs --tail 100 my-nginx

# Show logs with timestamps
docker logs -t my-nginx

# Show logs since specific time
docker logs --since 2024-01-01T00:00:00 my-nginx
docker logs --since 30m my-nginx    # Last 30 minutes
```

### Executing Commands Inside Running Container

```bash
# Run a command in a running container
docker exec my-nginx cat /etc/nginx/nginx.conf

# Open interactive shell
docker exec -it my-nginx bash
docker exec -it my-nginx sh      # If bash not available (alpine)

# Run as specific user
docker exec -u root my-nginx whoami

# Set working directory
docker exec -w /etc/nginx my-nginx ls -la

# Set environment variable for the command
docker exec -e MY_VAR=hello my-nginx env
```

### Inspecting Container Details

```bash
# Full container details (JSON)
docker inspect my-nginx

# Get specific fields
docker inspect --format '{{.NetworkSettings.IPAddress}}' my-nginx
docker inspect --format '{{.State.Status}}' my-nginx
docker inspect --format '{{json .Config.Env}}' my-nginx
docker inspect --format '{{.HostConfig.RestartPolicy}}' my-nginx

# See container resource usage (real-time)
docker stats
docker stats my-nginx

# Output:
# CONTAINER ID   NAME       CPU %   MEM USAGE / LIMIT   MEM %   NET I/O   BLOCK I/O
# abc123         my-nginx   0.02%   5.5MiB / 7.7GiB     0.07%   1.2kB     0B

# See processes running inside container
docker top my-nginx

# See filesystem changes (compared to image)
docker diff my-nginx
# Output:
# C /var     (Changed)
# A /var/log/nginx/access.log  (Added)
# D /tmp/old-file  (Deleted)
```

### Copying Files

```bash
# Copy FROM container to host
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf

# Copy FROM host to container
docker cp ./custom.conf my-nginx:/etc/nginx/nginx.conf

# Copy entire directory
docker cp my-nginx:/var/log/nginx/ ./logs/
```

---

## 🔄 Restart Policies

```
┌──────────────────────────────────────────────────────────────────┐
│                 RESTART POLICIES                                  │
│                                                                   │
│  Policy              │ Behavior                                  │
│  ────────────────────┼──────────────────────────────────────     │
│  no                  │ Never restart (default)                   │
│  on-failure          │ Restart only if exit code ≠ 0             │
│  on-failure:5        │ Max 5 restart attempts                    │
│  always              │ Always restart (survives daemon restart)  │
│  unless-stopped      │ Like always, but not if manually stopped  │
│                                                                   │
│  Production recommendations:                                     │
│  ├── Web servers  → always or unless-stopped                    │
│  ├── Workers      → on-failure:5                                │
│  ├── One-off jobs → no                                          │
│  └── Dev/test     → no                                          │
└──────────────────────────────────────────────────────────────────┘
```

```bash
# Set restart policy
docker run -d --restart=always nginx

# Update restart policy on existing container
docker update --restart=unless-stopped my-nginx
```

---

## 🔑 Environment Variables

```bash
# Pass single env var
docker run -d -e DB_HOST=localhost -e DB_PORT=5432 myapp

# Pass from file
# .env file:
# DB_HOST=localhost
# DB_PORT=5432
# DB_USER=admin
docker run -d --env-file .env myapp

# See env vars in running container
docker exec my-container env
docker inspect --format '{{json .Config.Env}}' my-container
```

---

## 🏢 Resource Constraints

```bash
# Memory limits
docker run -d --memory=256m nginx          # Hard limit: 256MB
docker run -d --memory=256m --memory-swap=512m nginx  # Swap: 512MB total
docker run -d --memory=256m --oom-kill-disable nginx   # Don't kill on OOM (dangerous!)

# CPU limits
docker run -d --cpus=1.5 nginx             # 1.5 CPU cores
docker run -d --cpu-shares=512 nginx        # Relative weight (default: 1024)
docker run -d --cpuset-cpus="0,1" nginx     # Only use CPU 0 and 1

# Combined
docker run -d --memory=256m --cpus=0.5 --name limited nginx

# Update limits on running container
docker update --memory=512m --cpus=1.0 limited

# View current usage
docker stats limited
```

```
┌──────────────────────────────────────────────────────────────────┐
│  WHAT HAPPENS WHEN LIMITS ARE HIT?                               │
│                                                                   │
│  Memory limit exceeded → Container is KILLED (OOMKilled)         │
│  CPU limit exceeded    → Container is THROTTLED (not killed)     │
│                                                                   │
│  This is same behavior as Kubernetes!                            │
│  (Because K8s uses the same cgroups mechanism)                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🧹 Cleanup Commands

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune          # Dangling only
docker image prune -a       # All unused

# Remove unused volumes
docker volume prune

# Remove unused networks
docker network prune

# NUCLEAR OPTION: Remove EVERYTHING unused
docker system prune         # Containers + images + networks
docker system prune -a      # + ALL unused images
docker system prune --volumes  # + volumes too

# Check disk usage
docker system df

# Output:
# TYPE            TOTAL   ACTIVE  SIZE    RECLAIMABLE
# Images          10      3       2.5GB   1.8GB (72%)
# Containers      5       2       50MB    30MB (60%)
# Local Volumes   8       3       1.2GB   800MB (66%)
# Build Cache     15      0       500MB   500MB (100%)
```

---

## 📊 Container States Reference

| State | `docker ps` shows | Meaning |
|-------|-------------------|---------|
| **Created** | Shows with `-a` only | Created but never started |
| **Running** | `Up 5 minutes` | Currently running |
| **Paused** | `Up 5 min (Paused)` | Processes frozen |
| **Restarting** | `Restarting` | Between stop and start |
| **Exited** | `Exited (0)` | Stopped (exit code shown) |
| **Dead** | `Dead` | Partially removed (error during rm) |

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success (clean exit) |
| `1` | Application error |
| `125` | Docker daemon error |
| `126` | Command cannot be invoked |
| `127` | Command not found |
| `137` | Killed by SIGKILL (OOMKilled or `docker kill`) |
| `143` | Killed by SIGTERM (graceful `docker stop`) |

---

## 🧠 Memory Shortcuts for This Chapter

### Container lifecycle: **"CRSD"**
```
C = Create (from image)
R = Run (start the process)
S = Stop (graceful shutdown)
D = Delete (remove container)
```

### Debugging trio: **"LET"**
```
L = Logs (docker logs)
E = Exec (docker exec -it ... bash)
T = Top/Stats (docker top / docker stats)
```

### Run flags: **"DPEN"** (Deep-N)
```
D = -d (detached/background)
P = -p (port mapping)
E = -e (environment variables)
N = --name (name your container)
```

---

## ❓ Quick Quiz

1. What is the difference between `docker stop` and `docker kill`?
2. What does `-d` flag do in `docker run`?
3. What happens when a container exceeds its memory limit?
4. What is the difference between `docker exec` and `docker attach`?
5. How do you check what files changed inside a running container?

<details>
<summary>Click for Answers</summary>

1. `stop` sends SIGTERM (graceful, waits 10s) then SIGKILL. `kill` sends SIGKILL immediately (no graceful shutdown).
2. Runs container in detached mode (background). Without `-d`, the container runs in the foreground attached to your terminal.
3. The container is killed (OOMKilled, exit code 137). For CPU limits, it's throttled but not killed.
4. `exec` starts a NEW process in the container. `attach` connects to the container's PID 1 (main process). Use `exec` for debugging, `attach` to see main process output.
5. `docker diff <container>` shows all filesystem changes (Added, Changed, Deleted) compared to the image.

</details>

---

**← Previous: [04 - Images Deep Dive](./04-images-deep-dive.md)** | **Next: [06 - Dockerfile Mastery](./06-dockerfile-mastery.md)** ➡️
