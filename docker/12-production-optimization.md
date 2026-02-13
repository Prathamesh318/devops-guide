# ⚡ Chapter 12: Production Optimization

> **"Development Docker is easy. Production Docker requires understanding performance, monitoring, logging, and reliability."**

---

## 🎯 Image Size Optimization

### The Size Problem

```
┌──────────────────────────────────────────────────────────────────┐
│  WHY IMAGE SIZE MATTERS IN PRODUCTION                            │
│                                                                   │
│  Smaller image = Faster deploys                                  │
│  Smaller image = Less bandwidth                                  │
│  Smaller image = Smaller attack surface                          │
│  Smaller image = Less storage cost                               │
│  Smaller image = Faster autoscaling (K8s)                        │
│                                                                   │
│  Before optimization:  python:3.11        → 920MB               │
│  After optimization:   python:3.11-slim   → 150MB (-83%)        │
│  After multi-stage:    custom slim build   → 80MB (-91%)         │
│  Using distroless:     distroless/python  → 50MB (-95%)          │
└──────────────────────────────────────────────────────────────────┘
```

### Optimization Techniques

```dockerfile
# 1. Use slim/alpine base images
FROM python:3.11-slim     # Instead of python:3.11

# 2. Combine RUN commands (fewer layers)
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# 3. Use --no-install-recommends (apt)
RUN apt-get install -y --no-install-recommends python3

# 4. Clean up in SAME layer
RUN pip install --no-cache-dir -r requirements.txt
#    --no-cache-dir saves space!

# 5. Multi-stage builds (only ship the output)
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html

# 6. Use .dockerignore to exclude unnecessary files

# 7. Order Dockerfile by change frequency
# (rarely-changed instructions first for better caching)
```

---

## 📊 Logging Best Practices

```
┌──────────────────────────────────────────────────────────────────┐
│              DOCKER LOGGING ARCHITECTURE                          │
│                                                                   │
│  App writes to:  stdout / stderr                                │
│       │                                                           │
│       ▼                                                           │
│  Docker captures output via log driver                          │
│       │                                                           │
│       ├── json-file (default — stored on disk)                  │
│       ├── syslog (send to syslog server)                        │
│       ├── journald (systemd journal)                             │
│       ├── fluentd (send to Fluentd)                             │
│       ├── awslogs (send to CloudWatch)                          │
│       ├── gcplogs (send to Google Cloud Logging)                │
│       └── none (discard logs)                                   │
│                                                                   │
│  RULE: Apps should log to stdout/stderr, NOT to files!           │
│  Let Docker handle log collection, rotation, and shipping.       │
└──────────────────────────────────────────────────────────────────┘
```

### Log Configuration

```bash
# View logs
docker logs --tail 100 -f mycontainer

# Set log driver per container
docker run -d \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    myapp

# ⚠️ Without log rotation, logs can fill your disk!
# ALWAYS set max-size and max-file in production!
```

### Daemon-Level Log Configuration

```json
// /etc/docker/daemon.json
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3",
        "compress": "true"
    }
}
```

---

## 📈 Monitoring & Health Checks

### Container Health Checks

```dockerfile
# In Dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=15s \
    CMD curl -f http://localhost:8080/health || exit 1
```

```bash
# Check status
docker ps     # STATUS column shows (healthy) / (unhealthy)
docker inspect --format='{{json .State.Health}}' mycontainer
```

### Resource Monitoring

```bash
# Real-time stats for all containers
docker stats

# Output:
# CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O   BLOCK I/O
# web         2.5%    50MiB / 256MiB      19%     1.2GB     50MB
# db          15%     200MiB / 512MiB     39%     500MB     2.1GB
# redis       0.3%    10MiB / 128MiB      7%      100MB     5MB

# JSON output for scripts
docker stats --format '{{json .}}' --no-stream mycontainer

# Prometheus metrics endpoint
# Use cAdvisor for container metrics:
docker run -d \
    --name cadvisor \
    -v /:/rootfs:ro \
    -v /var/run:/var/run:ro \
    -v /sys:/sys:ro \
    -v /var/lib/docker/:/var/lib/docker:ro \
    -p 8082:8080 \
    gcr.io/cadvisor/cadvisor:latest
```

---

## ⚡ Performance Tuning

### Container Resource Limits

```bash
# Memory
docker run -d --memory=256m --memory-swap=512m myapp

# CPU
docker run -d --cpus=1.5 myapp                  # 1.5 cores
docker run -d --cpu-shares=512 myapp             # Relative weight

# I/O
docker run -d --device-read-bps /dev/sda:10mb myapp
docker run -d --device-write-bps /dev/sda:10mb myapp
```

### Storage Driver Optimization

```json
// /etc/docker/daemon.json
{
    "storage-driver": "overlay2",
    "storage-opts": [
        "overlay2.size=10G"
    ]
}
```

### Network Performance

```bash
# Host network (no overhead, best performance)
docker run --network host myapp

# DNS caching (reduce DNS lookups)
docker run --dns 127.0.0.1 myapp  # Local DNS cache
```

---

## 🔄 Graceful Shutdown

```
┌──────────────────────────────────────────────────────────────────┐
│  GRACEFUL SHUTDOWN — Why It Matters                               │
│                                                                   │
│  docker stop myapp                                               │
│       │                                                           │
│       ▼                                                           │
│  Sends SIGTERM to PID 1 in container                            │
│       │                                                           │
│       ▼                                                           │
│  App should:                                                     │
│  ├── Stop accepting new requests                                │
│  ├── Finish processing current requests                         │
│  ├── Close database connections                                 │
│  ├── Flush logs and caches                                      │
│  └── Exit with code 0                                           │
│       │                                                           │
│       ▼                                                           │
│  If app doesn't exit in 10s → Docker sends SIGKILL (forced)     │
│                                                                   │
│  CRITICAL: Use exec form in CMD/ENTRYPOINT!                      │
│  CMD ["python", "app.py"]    ✅ PID 1 = python (gets SIGTERM)   │
│  CMD python app.py           ❌ PID 1 = /bin/sh (doesn't pass!) │
└──────────────────────────────────────────────────────────────────┘
```

```bash
# Custom stop timeout
docker stop -t 30 myapp    # Wait 30s before SIGKILL

# Custom stop signal
docker run --stop-signal SIGQUIT myapp
```

---

## 🏭 Production Compose File

```yaml
# compose.prod.yml — Production-ready configuration
services:
  api:
    image: myregistry/api:v2.1.3    # Specific version, NEVER :latest
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    networks:
      - backend
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    restart: unless-stopped
    volumes:
      - pg-data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 1G
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - backend

volumes:
  pg-data:

networks:
  backend:
    driver: bridge
```

---

## 🧠 Memory Shortcuts for This Chapter

### Production checklist: **"LMRSH"**
```
L = Logging (configure drivers, rotation)
M = Monitoring (health checks, stats, cAdvisor)
R = Resources (memory/CPU limits)
S = Size (optimize image, multi-stage)
H = Health (HEALTHCHECK instruction, graceful shutdown)
```

---

## ❓ Quick Quiz

1. Why should applications log to stdout/stderr instead of files?
2. What happens if you don't configure log rotation?
3. What is the difference between memory limits and CPU limits when exceeded?
4. Why is exec form important for CMD/ENTRYPOINT in production?
5. What monitoring tools can you use for Docker containers?

<details>
<summary>Click for Answers</summary>

1. Docker captures stdout/stderr via log drivers, enabling centralized log management, rotation, and shipping to log aggregators.
2. Log files grow indefinitely, eventually filling the disk and crashing the host. Always set max-size and max-file.
3. Memory limit exceeded → container killed (OOMKilled). CPU limit exceeded → container throttled (slowed, not killed).
4. Exec form makes your app PID 1, so it receives SIGTERM for graceful shutdown. Shell form makes /bin/sh PID 1, which doesn't forward signals.
5. docker stats (built-in), cAdvisor (Prometheus metrics), Prometheus + Grafana (dashboards), Datadog, New Relic.

</details>

---

**← Previous: [11 - Security Best Practices](./11-security-best-practices.md)** | **Next: [13 - Hands-On Projects](./13-hands-on-projects.md)** ➡️
