# 🖼️ Chapter 4: Docker Images Deep Dive

> **"An image is to a container what a class is to an object — a blueprint from which you create running instances."**

---

## 🎯 What is a Docker Image?

```
┌──────────────────────────────────────────────────────────────────┐
│                    DOCKER IMAGE                                   │
│                                                                   │
│  An image is a READ-ONLY template that contains:                 │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  📁 A minimal filesystem (OS libraries)                    │  │
│  │  📦 Application runtime (Python, Node, Java, etc.)        │  │
│  │  📄 Application code                                       │  │
│  │  📋 Dependencies (pip packages, npm modules, etc.)         │  │
│  │  ⚙️  Configuration files                                   │  │
│  │  🔧 Environment variables                                  │  │
│  │  📌 Default command to run                                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Images are IMMUTABLE — once built, they never change.           │
│  Want to change it? Build a NEW image.                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📋 Image Naming Convention

```
┌──────────────────────────────────────────────────────────────────┐
│  IMAGE NAME FORMAT:                                               │
│                                                                   │
│  [registry/][namespace/]repository[:tag][@digest]                │
│                                                                   │
│  Examples:                                                        │
│  ─────────────────────────────────────────────────────────────    │
│  nginx                          (= docker.io/library/nginx:latest)│
│  nginx:1.25                     (specific version)               │
│  nginx:alpine                   (variant - smaller image)         │
│  myuser/myapp:v2.1              (user namespace)                 │
│  ghcr.io/myorg/myapp:latest     (GitHub Container Registry)     │
│  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1             │
│                                 (AWS ECR)                        │
│                                                                   │
│  TAGGING BEST PRACTICES:                                         │
│  ❌ myapp:latest    (ambiguous — what version is this?)          │
│  ✅ myapp:v2.1.3    (semantic versioning)                        │
│  ✅ myapp:abc1234   (git commit SHA)                             │
│  ✅ myapp:v2.1.3-abc1234 (version + commit — BEST!)             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Essential Image Commands

### Pulling Images

```bash
# Pull latest nginx
docker pull nginx
# Same as: docker pull docker.io/library/nginx:latest

# Pull specific version
docker pull nginx:1.25

# Pull from different registry
docker pull ghcr.io/myorg/myapp:v1

# Pull for specific platform
docker pull --platform linux/arm64 nginx:1.25
```

### Listing Images

```bash
# List all local images
docker images
# OR
docker image ls

# Output:
# REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
# nginx        latest    a6bd71f48f68   2 days ago    187MB
# nginx        alpine    1e5d5f3a28f5   2 days ago    43MB
# ubuntu       22.04     3b418d7b466a   1 week ago    77MB

# List with filtering
docker images --filter "dangling=true"    # Untagged images
docker images --filter "reference=nginx"  # Only nginx images
docker images --format "{{.Repository}}:{{.Tag}} {{.Size}}"

# List image IDs only
docker images -q
```

### Inspecting Images

```bash
# Detailed image info (JSON)
docker image inspect nginx:latest

# Get specific fields
docker image inspect nginx --format '{{.Os}}'
docker image inspect nginx --format '{{.Config.ExposedPorts}}'
docker image inspect nginx --format '{{.RootFS.Layers}}'

# See image history (how it was built — each layer)
docker image history nginx:latest

# Output shows each layer:
# IMAGE         CREATED       CREATED BY                                     SIZE
# a6bd71f48f68  2 days ago    CMD ["nginx" "-g" "daemon off;"]               0B
# <missing>     2 days ago    EXPOSE map[80/tcp:{}]                          0B
# <missing>     2 days ago    COPY file:xxx in /etc/nginx/nginx.conf         1.21kB
# <missing>     2 days ago    RUN /bin/sh -c apt-get update && apt-get...    85.8MB
# <missing>     2 days ago    /bin/sh -c #(nop) ADD file:xxx in /            77.8MB
```

### Tagging Images

```bash
# Tag an existing image with a new name
docker tag nginx:latest myregistry/nginx:v1.0

# Tag for different registry
docker tag myapp:latest 123456.dkr.ecr.us-east-1.amazonaws.com/myapp:v1

# Multiple tags for same image (common in CI/CD)
docker tag myapp:latest myapp:v2.1.3
docker tag myapp:latest myapp:v2.1.3-abc1234
docker tag myapp:latest myapp:stable
```

### Removing Images

```bash
# Remove specific image
docker rmi nginx:latest
# OR
docker image rm nginx:latest

# Remove by image ID
docker rmi a6bd71f48f68

# Force remove (even if containers exist)
docker rmi -f nginx:latest

# Remove ALL unused images (not used by any container)
docker image prune

# Remove ALL images (nuclear option)
docker image prune -a

# Remove dangling images (untagged leftovers from builds)
docker image prune --filter "dangling=true"
```

---

## 📊 Understanding Image Layers

```
┌──────────────────────────────────────────────────────────────────┐
│              LAYER SHARING BETWEEN IMAGES                        │
│                                                                   │
│   Image: myapp-frontend                Image: myapp-backend      │
│   ┌────────────────────┐               ┌────────────────────┐   │
│   │ Layer 5: React app │               │ Layer 5: Python app│   │
│   ├────────────────────┤               ├────────────────────┤   │
│   │ Layer 4: npm deps  │               │ Layer 4: pip deps  │   │
│   ├────────────────────┤               ├────────────────────┤   │
│   │ Layer 3: Node.js   │               │ Layer 3: Python    │   │
│   ├────────────────────┤               ├────────────────────┤   │
│   │ Layer 2: apt pkgs  │ ◄── SHARED ──►│ Layer 2: apt pkgs  │   │
│   ├────────────────────┤               ├────────────────────┤   │
│   │ Layer 1: Ubuntu    │ ◄── SHARED ──►│ Layer 1: Ubuntu    │   │
│   └────────────────────┘               └────────────────────┘   │
│                                                                   │
│   Layers 1 & 2 exist ONCE on disk, shared by both images!       │
│   Total disk: NOT (200MB + 200MB) = 400MB                       │
│   Actual disk: (200MB + shared layers) ≈ 280MB                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🏷️ Image Variants — Choosing the Right Base

```
┌──────────────────────────────────────────────────────────────────┐
│              COMMON IMAGE VARIANTS                                │
│                                                                   │
│  Tag          │ Base OS      │ Size    │ Use Case                │
│  ─────────────┼──────────────┼─────────┼─────────────────────    │
│  :latest      │ Debian       │ Large   │ Development             │
│  :bookworm    │ Debian 12    │ Large   │ Specific Debian version │
│  :slim        │ Debian (min) │ Medium  │ Smaller production      │
│  :alpine      │ Alpine Linux │ Small   │ Minimal production      │
│  :bullseye    │ Debian 11    │ Large   │ Older Debian version    │
│  :windowsservercore │ Windows │ Huge   │ Windows containers      │
│                                                                   │
│  Example sizes for Python:                                       │
│  python:3.11            → ~920MB  (full Debian)                  │
│  python:3.11-slim       → ~150MB  (minimal Debian)               │
│  python:3.11-alpine     → ~50MB   (Alpine Linux)                 │
│                                                                   │
│  Recommendation:                                                  │
│  Development → :slim (good balance)                              │
│  Production  → :slim or :alpine (smaller = faster deploys)       │
│  Debugging   → :latest (full tools available)                    │
│                                                                   │
│  ⚠️ Alpine gotchas:                                              │
│  • Uses musl libc instead of glibc (some C libraries may break) │
│  • Uses apk instead of apt                                      │
│  • Fewer pre-installed tools                                    │
│  • Some Python packages need compilation (slower builds)         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔎 Searching for Images

```bash
# Search Docker Hub from CLI
docker search nginx
docker search --filter is-official=true nginx
docker search --filter stars=100 python

# Better: Use Docker Hub website
# https://hub.docker.com
# Check: Official image ✅, Verified publisher ✅, Download count, Stars

# Check image for vulnerabilities (Docker Scout)
docker scout cves nginx:latest
docker scout quickview nginx:latest
```

### How to Choose the Right Image

```
1. Prefer OFFICIAL images (maintained by Docker + project teams)
   ✅ nginx, postgres, redis, python, node
   ❌ random-user/nginx-custom

2. Prefer VERIFIED PUBLISHER images
   ✅ bitnami/postgresql
   ❌ unknown-user/postgresql-hack

3. Check the Dockerfile
   Docker Hub → image page → "How to use this image"
   GitHub → see how the official Dockerfile is built

4. Check vulnerability scans
   docker scout cves <image>

5. Use specific tags, NEVER :latest in production
   ✅ postgres:15.4-alpine
   ❌ postgres:latest
```

---

## 💾 Saving and Loading Images (Offline Transfer)

```bash
# Save image to a tar file (for offline transfer)
docker save nginx:latest -o nginx-latest.tar
docker save nginx:latest | gzip > nginx-latest.tar.gz

# Load image from tar file
docker load -i nginx-latest.tar
docker load < nginx-latest.tar.gz

# Export container filesystem (NOT an image — flat filesystem)
docker export <container-id> -o container.tar

# Import filesystem as image
docker import container.tar myimage:imported
```

```
SAVE vs EXPORT:
─────────────────────────────────────────────
docker save    → Saves IMAGE with all layers and metadata
                  Use for: Moving images between machines
                  
docker export  → Saves CONTAINER filesystem as flat tar
                  Use for: Quick backup of container state
                  ⚠️ Loses all layer history and metadata
```

---

## 🔬 Inspecting Image Layers with `dive`

```bash
# Install dive (image layer explorer)
# https://github.com/wagoodman/dive

# Analyze an image
dive nginx:latest

# You'll see:
# ┌─────────────────────────────────────────────────┐
# │ Layer 1: 77MB   Ubuntu base files               │
# │ Layer 2: 85MB   Nginx installed                  │
# │ Layer 3: 1.2KB  Config copied                    │
# │ Layer 4: 0B     CMD instruction                  │
# ├─────────────────────────────────────────────────┤
# │ Image efficiency score: 97%                      │
# │ Wasted space: 2.3MB                              │
# └─────────────────────────────────────────────────┘

# CI mode (fails if image is inefficient)
dive nginx:latest --ci
```

---

## 🧠 Memory Shortcuts for This Chapter

### Image commands: **"PLIRT"**
```
P = Pull (download from registry)
L = List (docker images)
I = Inspect (see metadata)
R = Remove (docker rmi)
T = Tag (rename/add tags)
```

### Image variants: **"Big → Slim → Alpine"**
```
Full (:latest)     = Everything included, big, good for dev
Slim (:slim)       = Just enough, medium, good for prod
Alpine (:alpine)   = Bare minimum, tiny, best for prod (but watch compatibility)
```

---

## ❓ Quick Quiz

1. What is the difference between an image and a container?
2. Why should you never use `:latest` in production?
3. What are image layers and why are they important?
4. What is the difference between `docker save` and `docker export`?
5. When would you choose `alpine` over `slim`?

<details>
<summary>Click for Answers</summary>

1. Image = read-only blueprint; Container = running instance of an image with a writable layer on top.
2. `:latest` is mutable — it points to whatever was pushed last. You could get different versions on different servers. Use specific tags for reproducibility.
3. Layers are stacked filesystem snapshots. They're important because they are shared between images (save disk) and cached during builds (save time).
4. `save` preserves all layers and metadata (for moving images); `export` flattens container filesystem (loses history).
5. When you need the smallest possible image and your dependencies are compatible with musl libc. Use `slim` when you need glibc compatibility.

</details>

---

**← Previous: [03 - Installation & Setup](./03-installation-setup.md)** | **Next: [05 - Containers Lifecycle](./05-containers-lifecycle.md)** ➡️
