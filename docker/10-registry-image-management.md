# 🏪 Chapter 10: Registry & Image Management

> **"Docker Hub is just the beginning — production teams need private registries for control, security, and speed."**

---

## 🎯 What is a Container Registry?

```
┌──────────────────────────────────────────────────────────────────┐
│                 CONTAINER REGISTRY                                │
│                                                                   │
│  A registry is like GitHub but for Docker images.                │
│                                                                   │
│  GitHub:  Source Code  → git push → Repository                  │
│  Docker:  Image        → docker push → Registry                 │
│                                                                   │
│  Registry     │ Type     │ Provider                              │
│  ─────────────┼──────────┼────────────────────────────────       │
│  Docker Hub   │ Public   │ Docker Inc                            │
│  GHCR         │ Public   │ GitHub (ghcr.io)                     │
│  ECR          │ Private  │ AWS                                   │
│  GCR / AR     │ Private  │ Google Cloud                          │
│  ACR          │ Private  │ Azure                                 │
│  Harbor       │ Private  │ Self-hosted (CNCF)                   │
│  JFrog        │ Private  │ JFrog Artifactory                    │
│  Quay.io      │ Public   │ Red Hat                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🐳 Docker Hub

### Push & Pull

```bash
# Login to Docker Hub
docker login
# Enter username and password

# Tag your image for Docker Hub
docker tag myapp:latest username/myapp:v1.0

# Push to Docker Hub
docker push username/myapp:v1.0

# Pull someone else's image
docker pull username/myapp:v1.0

# Logout
docker logout
```

### Docker Hub Concepts

```
Organization:  mycompany/
Repository:    mycompany/web-app
Tags:          mycompany/web-app:v1.0
               mycompany/web-app:v1.1
               mycompany/web-app:latest

Free tier:  1 private repo, unlimited public repos
Pro tier:   5 private repos
Team tier:  Unlimited private repos
```

---

## ☁️ Cloud Registry Setup

### AWS ECR

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin \
    123456789.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name myapp

# Tag and push
docker tag myapp:v1 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
```

### GitHub Container Registry (GHCR)

```bash
# Login with GitHub token
echo $GITHUB_TOKEN | docker login ghcr.io -u username --password-stdin

# Tag and push
docker tag myapp:v1 ghcr.io/username/myapp:v1
docker push ghcr.io/username/myapp:v1
```

### Google Artifact Registry

```bash
# Configure Docker for Google Cloud
gcloud auth configure-docker us-central1-docker.pkg.dev

# Tag and push
docker tag myapp:v1 us-central1-docker.pkg.dev/project-id/repo-name/myapp:v1
docker push us-central1-docker.pkg.dev/project-id/repo-name/myapp:v1
```

---

## 🏠 Self-Hosted Private Registry

```bash
# Run your own registry (it's just a Docker container!)
docker run -d -p 5000:5000 \
    --name registry \
    --restart=always \
    -v registry-data:/var/lib/registry \
    registry:2

# Push to local registry
docker tag myapp:v1 localhost:5000/myapp:v1
docker push localhost:5000/myapp:v1

# Pull from local registry
docker pull localhost:5000/myapp:v1

# List repositories (API)
curl http://localhost:5000/v2/_catalog

# With TLS and authentication
docker run -d -p 5000:5000 \
    --name registry \
    -v $(pwd)/certs:/certs \
    -v $(pwd)/auth:/auth \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
    -e REGISTRY_AUTH=htpasswd \
    -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
    registry:2
```

---

## 🏷️ Image Tagging Strategies

```
┌──────────────────────────────────────────────────────────────────┐
│              IMAGE TAGGING BEST PRACTICES                         │
│                                                                   │
│  Strategy         │ Example Tags          │ Use Case             │
│  ─────────────────┼───────────────────────┼────────────────────  │
│  Semantic Version │ v1.2.3                │ Release versions     │
│  Git SHA          │ abc1234               │ Exact commit mapping │
│  Branch           │ main, develop         │ CI/CD pipelines      │
│  Date-based       │ 2024-01-15            │ Nightly builds       │
│  Combined         │ v1.2.3-abc1234        │ Best of both worlds  │
│                                                                   │
│  ❌ NEVER in production:                                         │
│  :latest    → Ambiguous, different on each machine              │
│  :stable    → Meaningless without version                       │
│                                                                   │
│  ✅ ALWAYS in production:                                        │
│  :v2.1.3             → You know exactly what's running          │
│  :v2.1.3-abc1234     → Version + exact build commit             │
│  :v2.1.3-20240115    → Version + build date                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔄 CI/CD Image Pipeline

```
┌──────────────────────────────────────────────────────────────────┐
│              CI/CD IMAGE PIPELINE                                 │
│                                                                   │
│  Developer pushes code                                           │
│       │                                                           │
│       ▼                                                           │
│  CI/CD Pipeline triggers                                         │
│       │                                                           │
│       ├── 1. Run tests                                           │
│       ├── 2. Build image: docker build -t myapp:${GIT_SHA} .    │
│       ├── 3. Scan for vulnerabilities: docker scout cves myapp   │
│       ├── 4. Tag with version + SHA                              │
│       │      docker tag myapp:${GIT_SHA} myapp:v${VERSION}      │
│       ├── 5. Push to registry: docker push myapp:v${VERSION}    │
│       └── 6. Deploy to staging/production                        │
│                                                                   │
│  Image tags created:                                             │
│  myapp:abc1234         (commit SHA)                              │
│  myapp:v2.1.3          (semantic version)                        │
│  myapp:v2.1.3-abc1234  (combined — BEST)                        │
│  myapp:latest          (only for convenience, never for deploy)  │
└──────────────────────────────────────────────────────────────────┘
```

### GitHub Actions Example

```yaml
# .github/workflows/docker-publish.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
```

---

## 🔐 Image Security Scanning

```bash
# Docker Scout (built-in)
docker scout cves nginx:latest
docker scout quickview nginx:latest
docker scout recommendations nginx:latest

# Trivy (popular open-source scanner)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image nginx:latest

# Snyk
docker scan nginx:latest   # (Docker Desktop integration)
```

---

## 🧠 Memory Shortcuts for This Chapter

### Registry workflow: **"LTPP"**
```
L = Login (authenticate to registry)
T = Tag (name image for registry)
P = Push (upload to registry)
P = Pull (download from registry)
```

### Tagging rules:
```
Development → :latest, :dev, :branch-name
Staging     → :v1.2.3-rc1, :staging
Production  → :v1.2.3 (ALWAYS specific version)
CI/CD       → :v1.2.3-abc1234 (version + commit)
```

---

## ❓ Quick Quiz

1. What is a container registry?
2. Why should you never use `:latest` in production?
3. What is the advantage of a self-hosted registry?
4. How does a CI/CD pipeline typically handle Docker images?
5. Name three ways to scan images for vulnerabilities.

<details>
<summary>Click for Answers</summary>

1. A storage and distribution service for container images, like Docker Hub, ECR, GHCR, or self-hosted with Harbor/registry:2.
2. `:latest` is mutable — different machines may pull different versions. You lose reproducibility and can't reliably rollback.
3. Full control over security, no rate limits, faster pulls (same network), compliance with data residency requirements.
4. Build image → test → scan for vulnerabilities → tag with version/commit → push to registry → deploy.
5. Docker Scout, Trivy, Snyk, Grype, Clair.

</details>

---

**← Previous: [09 - Docker Compose](./09-docker-compose.md)** | **Next: [11 - Security Best Practices](./11-security-best-practices.md)** ➡️
