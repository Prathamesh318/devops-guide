# 📝 Chapter 6: Dockerfile Mastery

> **"A Dockerfile is the recipe for your image. Master the recipe, and you control the entire container."**

---

## 🎯 What is a Dockerfile?

```
┌──────────────────────────────────────────────────────────────────┐
│                    DOCKERFILE                                     │
│                                                                   │
│  A text file with sequential instructions that tell Docker:      │
│                                                                   │
│  1. What base OS/runtime to start from                           │
│  2. What packages to install                                     │
│  3. What files to copy                                           │
│  4. What commands to run during build                            │
│  5. What command to run when container starts                    │
│                                                                   │
│  Dockerfile  ──(docker build)──►  Image  ──(docker run)──► Container │
│  (recipe)                      (blueprint)              (running) │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📋 All Dockerfile Instructions

| Instruction | Purpose | Layer? |
|-------------|---------|--------|
| `FROM` | Base image | Yes |
| `RUN` | Execute command during build | Yes |
| `COPY` | Copy files from host to image | Yes |
| `ADD` | Copy files + auto-extract archives + URL support | Yes |
| `WORKDIR` | Set working directory | Yes |
| `ENV` | Set environment variable | Yes |
| `ARG` | Build-time variable (NOT in final image) | No |
| `EXPOSE` | Document which ports the container listens on | No |
| `CMD` | Default command when container starts | No |
| `ENTRYPOINT` | Main command (not easily overridden) | No |
| `VOLUME` | Create a mount point for data | No |
| `USER` | Set user for RUN, CMD, ENTRYPOINT | No |
| `LABEL` | Add metadata to image | Yes (tiny) |
| `HEALTHCHECK` | Define how Docker checks container health | No |
| `SHELL` | Override default shell | No |
| `STOPSIGNAL` | Set signal to stop the container | No |

---

## 🔧 Instruction Deep Dive

### FROM — The Starting Point

```dockerfile
# EVERY Dockerfile must start with FROM
FROM ubuntu:22.04

# Multi-arch aware
FROM --platform=linux/amd64 python:3.11-slim

# Use scratch for ultra-minimal (no OS at all)
FROM scratch   # For statically compiled binaries (Go, Rust)
```

### RUN — Execute Commands During Build

```dockerfile
# Each RUN creates a NEW layer!

# ❌ BAD: Multiple RUN = multiple layers = bigger image
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ GOOD: Single RUN = single layer = smaller image
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3 \
        curl && \
    rm -rf /var/lib/apt/lists/*
# The cleanup in SAME RUN actually removes data from the layer
# If cleanup is in a separate RUN, the files still exist in the previous layer!
```

### COPY vs ADD

```dockerfile
# COPY — Simple, predictable, preferred ✅
COPY app.py /app/
COPY requirements.txt /app/
COPY . /app/                    # Copy everything (use .dockerignore!)
COPY --chown=1000:1000 . /app/  # Copy with ownership

# ADD — Extra features (use ONLY when needed)
ADD archive.tar.gz /app/        # Auto-extracts tar archives
ADD https://example.com/file /app/  # Download from URL (not recommended)

# RULE: Always use COPY unless you specifically need ADD's features
```

### WORKDIR — Set Working Directory

```dockerfile
# WORKDIR creates the directory if it doesn't exist
WORKDIR /app

# All following commands run from /app
COPY . .           # Copies to /app/
RUN pip install -r requirements.txt  # Runs in /app/
CMD ["python", "main.py"]           # Runs in /app/

# ❌ BAD: Using cd in RUN
RUN cd /app && pip install ...

# ✅ GOOD: Using WORKDIR
WORKDIR /app
RUN pip install ...
```

### ENV and ARG

```dockerfile
# ENV — Available during build AND in running container
ENV NODE_ENV=production
ENV DB_HOST=localhost DB_PORT=5432

# ARG — Available ONLY during build (not in container!)
ARG VERSION=1.0
ARG BUILD_DATE

# Usage during build:
RUN echo "Building version $VERSION"

# Build with different ARG:
# docker build --build-arg VERSION=2.0 .

# Common pattern: ARG → ENV
ARG APP_VERSION=1.0
ENV VERSION=$APP_VERSION    # Now available in container too
```

### CMD vs ENTRYPOINT

```
┌──────────────────────────────────────────────────────────────────┐
│              CMD vs ENTRYPOINT                                    │
│                                                                   │
│  CMD = Default command (easily overridden)                       │
│  ENTRYPOINT = Main command (hard to override)                    │
│                                                                   │
│  Example:                                                         │
│  ────────────────────────────────────────────                     │
│  CMD ["python", "app.py"]                                        │
│  docker run myapp              → python app.py                   │
│  docker run myapp bash         → bash (CMD overridden!)          │
│                                                                   │
│  ENTRYPOINT ["python", "app.py"]                                 │
│  docker run myapp              → python app.py                   │
│  docker run myapp --debug      → python app.py --debug           │
│                                                             (appended!)│
│                                                                   │
│  BEST PRACTICE: Combine both                                     │
│  ENTRYPOINT ["python"]    ← The executable                      │
│  CMD ["app.py"]           ← Default argument (overridable)      │
│                                                                   │
│  docker run myapp              → python app.py                   │
│  docker run myapp test.py      → python test.py                  │
└──────────────────────────────────────────────────────────────────┘
```

```dockerfile
# Shell form vs Exec form
# ─────────────────────────────────
# Shell form (runs via /bin/sh -c)
CMD python app.py                  # PID 1 = /bin/sh, NOT python!
                                   # ❌ SIGTERM goes to shell, not app

# Exec form (runs directly — preferred ✅)
CMD ["python", "app.py"]           # PID 1 = python
                                   # ✅ SIGTERM goes directly to python
```

### EXPOSE

```dockerfile
# EXPOSE is DOCUMENTATION — it does NOT actually publish the port!
EXPOSE 8080
EXPOSE 8080/tcp
EXPOSE 8080/udp

# You STILL need -p when running:
# docker run -p 8080:8080 myapp

# EXPOSE tells:
# • Developers what ports the app uses
# • docker run -P to auto-map these ports
```

### USER

```dockerfile
# Create non-root user and switch to it
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser    # All subsequent commands run as appuser

# Or use numeric IDs (more portable)
USER 1000:1000

# SECURITY: Never run containers as root in production!
```

### HEALTHCHECK

```dockerfile
# Tell Docker how to check if container is healthy
HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=10s \
    CMD curl -f http://localhost:8080/healthz || exit 1

# --interval    = Check every 30 seconds
# --timeout     = Wait max 5 seconds for response
# --retries     = Mark unhealthy after 3 failures
# --start-period = Grace period for app startup

# Check health status:
# docker ps → STATUS shows (healthy) or (unhealthy)
# docker inspect --format='{{.State.Health.Status}}' container
```

---

## 🏗️ Complete Dockerfile Examples

### Python Application

```dockerfile
# ── Stage: Production Image ──
FROM python:3.11-slim

# Labels
LABEL maintainer="dev@example.com"
LABEL version="1.0"

# Don't run as root
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set working directory
WORKDIR /app

# Install dependencies first (better layer caching!)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=5s \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

# Run the application
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

### Node.js Application

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files first (dependency caching)
COPY package.json package-lock.json ./
RUN npm ci --only=production    # ci = clean install (faster, reproducible)

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s \
    CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

### Go Application (Multi-Stage)

```dockerfile
# ── Stage 1: Build ──
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server .

# ── Stage 2: Production (FROM SCRATCH — no OS!) ──
FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]

# Final image size: ~10MB (vs ~300MB without multi-stage!)
```

---

## 🚀 Multi-Stage Builds — The Game Changer

```
┌──────────────────────────────────────────────────────────────────┐
│              MULTI-STAGE BUILD                                    │
│                                                                   │
│  Problem: Build tools make images HUGE                           │
│  Java: JDK (400MB) vs JRE (200MB)                               │
│  Node: node_modules with devDependencies (500MB) vs production   │
│  Go: Go compiler (300MB) vs compiled binary (10MB)               │
│                                                                   │
│  Solution: Use multiple FROM instructions                        │
│                                                                   │
│  Stage 1 (builder):           Stage 2 (production):              │
│  ┌─────────────────────┐     ┌──────────────────────┐           │
│  │ Full OS + tools     │     │ Minimal OS           │           │
│  │ Source code          │     │                      │           │
│  │ Compile/build        │     │ COPY --from=builder  │           │
│  │ All dev dependencies │     │   only the binary/   │           │
│  │ Test frameworks      │     │   compiled output    │           │
│  │                      │     │                      │           │
│  │ Size: 800MB          │     │ Size: 50MB ✅       │           │
│  └─────────────────────┘     └──────────────────────┘           │
│  (discarded!)                (final image!)                      │
└──────────────────────────────────────────────────────────────────┘
```

```dockerfile
# Java multi-stage example
# ── Build Stage ──
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline    # Cache dependencies
COPY src ./src
RUN mvn package -DskipTests      # Build JAR

# ── Production Stage ──
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
RUN addgroup -S app && adduser -S app -G app
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]

# Build stage: ~600MB (JDK + Maven + source + dependencies)
# Final image: ~150MB (JRE + JAR only)
```

---

## 🎯 Layer Caching — Speed Up Your Builds

```
┌──────────────────────────────────────────────────────────────────┐
│              DOCKER BUILD CACHE                                   │
│                                                                   │
│  Docker caches each layer. If the instruction AND all previous   │
│  layers haven't changed, Docker REUSES the cached layer.         │
│                                                                   │
│  ❌ BAD: Code changes bust ALL caches                            │
│  ┌──────────────────────────────────────┐                       │
│  │ FROM python:3.11                    │ ← cached              │
│  │ COPY . .                            │ ← CACHE BUSTED (code changed)│
│  │ RUN pip install -r requirements.txt │ ← must re-run ❌      │
│  │ CMD ["python", "app.py"]           │ ← must re-run ❌      │
│  └──────────────────────────────────────┘                       │
│                                                                   │
│  ✅ GOOD: Dependencies cached separately                        │
│  ┌──────────────────────────────────────┐                       │
│  │ FROM python:3.11                    │ ← cached              │
│  │ COPY requirements.txt .             │ ← cached (if unchanged)│
│  │ RUN pip install -r requirements.txt │ ← cached ✅           │
│  │ COPY . .                            │ ← rebuilt (code changed)│
│  │ CMD ["python", "app.py"]           │ ← rebuilt              │
│  └──────────────────────────────────────┘                       │
│                                                                   │
│  Rule: Copy files that change LEAST first,                       │
│        files that change MOST last.                              │
│                                                                   │
│  Deps change rarely → COPY first                                │
│  Code changes often → COPY last                                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📄 .dockerignore — Keep Your Build Clean

```
# .dockerignore (same syntax as .gitignore)
# Put this NEXT TO your Dockerfile

# Version control
.git
.gitignore

# Dependencies (will be installed in build)
node_modules
__pycache__
*.pyc
venv/
.venv/

# IDE
.vscode/
.idea/
*.swp
*.swo

# Build outputs
dist/
build/

# Docker
Dockerfile
docker-compose*.yml
.dockerignore

# Docs and tests
README.md
docs/
tests/
*.md

# Environment files (secrets!)
.env
.env.local
*.key
*.pem

# OS files
.DS_Store
Thumbs.db
```

**Why .dockerignore matters:**
- Smaller build context (faster `docker build`)
- Prevents accidentally copying secrets into images
- Avoids cache-busting from irrelevant file changes

---

## 🔨 Building Images

```bash
# Basic build
docker build -t myapp:v1 .

# Build with specific Dockerfile
docker build -t myapp:v1 -f Dockerfile.prod .

# Build with build arguments
docker build --build-arg VERSION=2.0 -t myapp:v2 .

# Build with no cache (force rebuild all layers)
docker build --no-cache -t myapp:v1 .

# Build specific stage only (for testing)
docker build --target builder -t myapp:builder .

# Build with progress output
docker build --progress=plain -t myapp:v1 .

# Build with BuildKit (faster, better caching)
DOCKER_BUILDKIT=1 docker build -t myapp:v1 .

# Multi-platform build (for ARM + AMD64)
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:v1 .
```

---

## 🧠 Memory Shortcuts for This Chapter

### Dockerfile order: **"FRoCoCE"** (Fro-Co-CE)
```
FR = FROM (base image)
R  = RUN (install dependencies)
Co = COPY deps first, then code
C  = CMD / ENTRYPOINT (startup command)
E  = EXPOSE (document ports)
```

### Build optimization: **"SLIM"**
```
S = Small base (alpine/slim)
L = Layer caching (deps before code)
I = Ignore files (.dockerignore)
M = Multi-stage (separate build from production)
```

---

## ❓ Quick Quiz

1. What's the difference between CMD and ENTRYPOINT?
2. Why should you copy `requirements.txt` before copying the full app code?
3. What is multi-stage build and why is it important?
4. What's the difference between shell form and exec form for CMD?
5. Why should you combine RUN commands with `&&`?

<details>
<summary>Click for Answers</summary>

1. CMD provides default arguments (easily overridden with `docker run ... newcmd`). ENTRYPOINT defines the main executable (arguments are appended).
2. Layer caching — if requirements.txt hasn't changed, Docker reuses the pip install layer. Copying all code first would bust the cache every time any code file changes.
3. Build stage has full tools (compilers, dev deps). Production stage copies ONLY the built artifacts. Result: much smaller images.
4. Shell form (`CMD python app.py`) runs via `/bin/sh -c`, so PID 1 is shell (signals don't reach app). Exec form (`CMD ["python", "app.py"]`) runs directly as PID 1 (signals work correctly).
5. Each RUN creates a layer. Files deleted in a later layer still exist in the previous layer (just hidden). Combining commands keeps the layer small.

</details>

---

**← Previous: [05 - Containers Lifecycle](./05-containers-lifecycle.md)** | **Next: [07 - Networking Deep Dive](./07-networking-deep-dive.md)** ➡️
