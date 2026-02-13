# 🔒 Chapter 11: Docker Security Best Practices

> **"Security isn't a feature — it's a requirement. A vulnerable container can compromise your entire infrastructure."**

---

## 🎯 Docker Security Layers

```
┌──────────────────────────────────────────────────────────────────┐
│              DOCKER SECURITY — Defense in Depth                   │
│                                                                   │
│  Layer 1: HOST SECURITY                                          │
│  ├── Keep Docker updated                                        │
│  ├── Restrict access to Docker socket                           │
│  └── Use rootless mode if possible                              │
│                                                                   │
│  Layer 2: IMAGE SECURITY                                         │
│  ├── Use official/verified base images                          │
│  ├── Scan for vulnerabilities                                   │
│  ├── Use minimal images (alpine/distroless)                     │
│  └── Don't include secrets in images                            │
│                                                                   │
│  Layer 3: BUILD SECURITY                                         │
│  ├── Multi-stage builds (no build tools in prod)                │
│  ├── .dockerignore (no secrets copied)                          │
│  └── Pin versions (FROM image:specific-tag)                     │
│                                                                   │
│  Layer 4: RUNTIME SECURITY                                       │
│  ├── Run as non-root USER                                       │
│  ├── Read-only filesystem                                       │
│  ├── Drop capabilities                                          │
│  ├── Resource limits (memory, CPU)                               │
│  └── Seccomp, AppArmor, SELinux profiles                        │
│                                                                   │
│  Layer 5: NETWORK SECURITY                                       │
│  ├── User-defined networks (isolation)                          │
│  ├── Don't expose unnecessary ports                             │
│  └── Never expose docker.sock                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 👤 Rule #1: Never Run as Root

```dockerfile
# ❌ BAD: Container runs as root (default!)
FROM node:20
COPY . /app
CMD ["node", "server.js"]
# PID 1 runs as root → if hacker escapes, they're root on host!

# ✅ GOOD: Create and use non-root user
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser
RUN chown -R appuser:appuser /app
# Switch to non-root
USER appuser
CMD ["node", "server.js"]
```

```bash
# Run with specific user at runtime
docker run --user 1000:1000 myapp

# Verify current user inside container
docker exec mycontainer whoami
docker exec mycontainer id
```

---

## 🛡️ Rule #2: Use Minimal Base Images

```
┌──────────────────────────────────────────────────────────────────┐
│  SMALLER IMAGE = SMALLER ATTACK SURFACE                          │
│                                                                   │
│  Image                  │ Size    │ Packages │ CVEs (typical)    │
│  ───────────────────────┼─────────┼──────────┼────────────────── │
│  ubuntu:22.04           │ 77MB    │ ~100     │ 20-40 ⚠️         │
│  python:3.11            │ 920MB   │ ~400     │ 50-100 🔴        │
│  python:3.11-slim       │ 150MB   │ ~50      │ 5-15 ⚡          │
│  python:3.11-alpine     │ 50MB    │ ~15      │ 1-5 ✅            │
│  gcr.io/distroless/python│ 50MB   │ ~5       │ 0-2 ✅✅         │
│  scratch (empty)        │ 0MB     │ 0        │ 0 ✅✅✅          │
│                                                                   │
│  Distroless = Google's ultra-minimal images                      │
│  No shell, no package manager, nothing except your app           │
│  Attackers can't even get a shell!                               │
└──────────────────────────────────────────────────────────────────┘
```

```dockerfile
# Distroless example (Google's minimal images)
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
# No shell, no package manager, no tools for attackers!
```

---

## 🔐 Rule #3: Never Put Secrets in Images

```dockerfile
# ❌ TERRIBLE: Secret in image layer (visible to anyone!)
FROM node:20
ENV API_KEY=sk-1234567890abcdef
COPY . /app
CMD ["node", "server.js"]

# ❌ BAD: Secret in ARG (visible in image history!)
ARG DB_PASSWORD=secret
RUN echo $DB_PASSWORD > /tmp/setup && rm /tmp/setup
# Even after deletion, it exists in the layer!

# ✅ GOOD: Pass secrets at runtime
FROM node:20-slim
COPY . /app
CMD ["node", "server.js"]
# docker run -e API_KEY=sk-123... myapp

# ✅ BETTER: Use Docker secrets (Swarm) or mount secrets
# docker run -v /path/to/secrets:/run/secrets:ro myapp

# ✅ BEST: Use BuildKit secrets for build-time secrets
FROM node:20-slim
RUN --mount=type=secret,id=npmrc,target=/app/.npmrc \
    npm ci --only=production
# Secret is NEVER stored in any image layer
```

---

## 🔒 Rule #4: Read-Only Filesystem

```bash
# Run container with read-only root filesystem
docker run --read-only --tmpfs /tmp --tmpfs /run nginx

# In Compose
services:
  web:
    image: nginx
    read_only: true
    tmpfs:
      - /tmp
      - /var/cache/nginx
      - /var/run
```

---

## 🎛️ Rule #5: Drop Capabilities

```bash
# Linux capabilities that containers get by default:
# CHOWN, DAC_OVERRIDE, FSETID, FOWNER, MKNOD, NET_RAW,
# SETGID, SETUID, SETFCAP, SETPCAP, NET_BIND_SERVICE, SYS_CHROOT,
# KILL, AUDIT_WRITE

# Drop ALL capabilities, add only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx

# ❌ NEVER use --privileged (gives ALL host access)
docker run --privileged myapp    # NEVER in production!
```

---

## 🔬 Rule #6: Scan Images for Vulnerabilities

```bash
# Docker Scout (built-in)
docker scout cves myapp:latest
docker scout quickview myapp:latest

# Trivy (comprehensive open-source scanner)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image myapp:latest

# Scan in CI/CD (fail build if critical CVEs found)
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest

# Snyk (commercial with free tier)
docker scan myapp:latest
```

---

## 🏗️ Secure Dockerfile Template

```dockerfile
# ─── Secure Production Dockerfile ───

# 1. Use specific version (not :latest)
FROM python:3.11-slim-bookworm

# 2. Labels for metadata
LABEL maintainer="team@company.com"
LABEL version="1.0"

# 3. Install only what's needed, clean up in same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl && \
    rm -rf /var/lib/apt/lists/*

# 4. Create non-root user EARLY
RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser

# 5. Set working directory
WORKDIR /app

# 6. Copy dependency files first (cache optimization)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 7. Copy application code with correct ownership
COPY --chown=appuser:appuser . .

# 8. Switch to non-root user
USER appuser

# 9. Expose port (documentation only)
EXPOSE 8000

# 10. Healthcheck
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 11. Use exec form for proper signal handling
ENTRYPOINT ["gunicorn"]
CMD ["--bind", "0.0.0.0:8000", "--workers", "2", "app:app"]
```

---

## 📊 Security Checklist

```
┌──────────────────────────────────────────────────────────────────┐
│              DOCKER SECURITY CHECKLIST                            │
│                                                                   │
│  IMAGE BUILDING:                                                 │
│  ☐ Use specific base image tags (not :latest)                   │
│  ☐ Use minimal base images (slim/alpine/distroless)             │
│  ☐ Multi-stage builds (no build tools in final image)           │
│  ☐ No secrets in Dockerfile (use BuildKit secrets)              │
│  ☐ .dockerignore excludes sensitive files                       │
│  ☐ Scan images for CVEs in CI/CD pipeline                       │
│                                                                   │
│  CONTAINER RUNTIME:                                              │
│  ☐ Run as non-root USER                                        │
│  ☐ Read-only root filesystem (--read-only)                      │
│  ☐ Drop all capabilities (--cap-drop ALL)                       │
│  ☐ Set memory and CPU limits                                    │
│  ☐ No --privileged flag                                         │
│  ☐ No docker.sock mount (unless absolutely needed)              │
│                                                                   │
│  NETWORK:                                                        │
│  ☐ Use user-defined networks (not default bridge)               │
│  ☐ Expose only necessary ports                                  │
│  ☐ Don't expose DB ports to host                                │
│  ☐ Use network segmentation (frontend/backend)                  │
│                                                                   │
│  HOST:                                                            │
│  ☐ Keep Docker Engine updated                                   │
│  ☐ Enable Docker Content Trust (image signing)                  │
│  ☐ Audit Docker daemon configuration                            │
│  ☐ Restrict Docker group membership                             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Memory Shortcuts for This Chapter

### Security layers: **"HIBRN"**
```
H = Host (update Docker, restrict socket)
I = Image (minimal, scanned, no secrets)
B = Build (multi-stage, .dockerignore)
R = Runtime (non-root, read-only, drop caps)
N = Network (isolation, minimal ports)
```

### Top 3 rules:
```
1. Never root → USER instruction
2. Never secrets in image → Runtime env vars
3. Never :latest → Pin specific versions
```

---

## ❓ Quick Quiz

1. Why should containers never run as root?
2. What are distroless images and why are they more secure?
3. How should you handle secrets in Docker?
4. What does `--cap-drop ALL` do?
5. Why is `--privileged` dangerous?

<details>
<summary>Click for Answers</summary>

1. If a container is compromised, root inside the container can lead to root on the host. Non-root limits the blast radius.
2. Distroless images contain only the app and its runtime dependencies — no shell, no package manager. Attackers can't even execute commands.
3. Never put secrets in Dockerfile/image. Pass at runtime via environment variables, mounted files, or Docker secrets. Use BuildKit `--mount=type=secret` for build-time secrets.
4. Removes all Linux capabilities from the container. You then add back only what's needed (principle of least privilege).
5. `--privileged` gives the container ALL host capabilities, access to ALL devices, and effectively removes all isolation. A compromised privileged container = compromised host.

</details>

---

**← Previous: [10 - Registry & Image Management](./10-registry-image-management.md)** | **Next: [12 - Production Optimization](./12-production-optimization.md)** ➡️
