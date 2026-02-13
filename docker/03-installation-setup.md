# 🛠️ Chapter 3: Installation & Setup

> **"You can't learn Docker by reading — install it, run it, break it, fix it."**

---

## 🖥️ Installation Options Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                 DOCKER INSTALLATION OPTIONS                       │
│                                                                   │
│  Your OS          │ Recommended Method                           │
│  ─────────────────┼──────────────────────────────────────────    │
│  Ubuntu/Debian    │ Docker Engine (apt install)                  │
│  CentOS/RHEL      │ Docker Engine (yum/dnf install)             │
│  Windows 10/11    │ Docker Desktop (WSL2 backend)               │
│  macOS            │ Docker Desktop (native)                      │
│  Cloud VM (Linux) │ Docker Engine (scripted install)            │
│                                                                   │
│  Docker Desktop = GUI + Docker Engine + Compose + Kubernetes    │
│  Docker Engine   = CLI + Daemon (server only, no GUI)           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🐧 Linux Installation (Ubuntu/Debian)

### Step 1: Uninstall Old Versions

```bash
# Remove old Docker versions if any
sudo apt-get remove docker docker-engine docker.io containerd runc
```

### Step 2: Install Prerequisites

```bash
# Update package index
sudo apt-get update

# Install required packages
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

### Step 3: Add Docker's Official GPG Key & Repository

```bash
# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository
echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/etc/apt/keyrings/docker.gpg] \
    https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 4: Install Docker Engine

```bash
# Update and install
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin

# What did we install?
# docker-ce           = Docker Engine (daemon)
# docker-ce-cli       = Docker CLI (client)
# containerd.io       = Container runtime
# docker-buildx-plugin = Next-gen build tool
# docker-compose-plugin = Multi-container tool
```

### Step 5: Post-Install Setup

```bash
# Add your user to docker group (run docker without sudo)
sudo usermod -aG docker $USER

# Activate changes (or log out and back in)
newgrp docker

# Enable Docker to start on boot
sudo systemctl enable docker
sudo systemctl enable containerd

# Start Docker
sudo systemctl start docker
```

### Step 6: Verify Installation

```bash
# Check Docker version
docker --version
# Docker version 24.0.7, build afdd53b

# Run hello-world test
docker run hello-world

# Expected output:
# Hello from Docker!
# This message shows that your installation appears to be working correctly.
```

---

## 🐧 Linux Installation (CentOS/RHEL/Fedora)

```bash
# Remove old versions
sudo yum remove docker docker-client docker-common docker-latest

# Install prerequisites
sudo yum install -y yum-utils

# Add Docker repository
sudo yum-config-manager --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin

# Start and enable
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group
sudo usermod -aG docker $USER
```

---

## 🏁 Quick Install Script (All Linux)

```bash
# Docker provides an official convenience script
# ONLY use this for development/testing, NOT production
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Post-install
sudo usermod -aG docker $USER
```

---

## 🪟 Windows Installation

### Prerequisites
- Windows 10/11 (64-bit, Build 19041+)
- WSL2 enabled
- Virtualization enabled in BIOS

### Steps

```
1. Enable WSL2:
   ────────────────────────────────────────────────
   Open PowerShell as Admin:
   > wsl --install
   > Restart computer

2. Download Docker Desktop:
   ────────────────────────────────────────────────
   Go to: https://www.docker.com/products/docker-desktop
   Download: Docker Desktop for Windows
   Run the installer

3. Configuration:
   ────────────────────────────────────────────────
   Settings → General → ☑ Use WSL 2 based engine
   Settings → Resources → WSL Integration → Enable for your distro

4. Verify:
   ────────────────────────────────────────────────
   Open PowerShell or CMD:
   > docker --version
   > docker run hello-world
```

---

## 🍎 macOS Installation

```
1. Download Docker Desktop:
   ────────────────────────────────────────────────
   Go to: https://www.docker.com/products/docker-desktop
   Download: Docker Desktop for Mac
   • Apple Silicon (M1/M2/M3) → Apple chip version
   • Intel Mac → Intel chip version

2. Install:
   ────────────────────────────────────────────────
   Drag Docker to Applications
   Open Docker Desktop
   Grant permissions when asked

3. Verify:
   ────────────────────────────────────────────────
   Open Terminal:
   $ docker --version
   $ docker run hello-world
```

---

## ✅ Verify Everything Works

After installation, run these commands:

```bash
# 1. Docker version (client + server)
docker version

# Output should show:
# Client:
#  Version:    24.x.x
# Server:
#  Engine:
#   Version:   24.x.x

# 2. Docker system info
docker info

# Shows: storage driver, runtime, OS, architecture, etc.

# 3. Run test container
docker run hello-world

# 4. Run interactive container
docker run -it ubuntu bash
# You're now INSIDE an Ubuntu container!
# Try: cat /etc/os-release
# Type: exit  (to leave)

# 5. Check Docker Compose
docker compose version
# Docker Compose version v2.x.x
```

---

## ⚙️ Essential Docker Configuration

### Daemon Configuration (`/etc/docker/daemon.json`)

```json
{
    "storage-driver": "overlay2",
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    },
    "default-address-pools": [
        {
            "base": "172.80.0.0/16",
            "size": 24
        }
    ],
    "dns": ["8.8.8.8", "8.8.4.4"],
    "live-restore": true,
    "userland-proxy": false
}
```

```
What each setting does:
─────────────────────────────────
storage-driver     → overlay2 (best for Linux, efficient layers)
log-driver         → json-file (default, logs in JSON format)
log-opts           → Prevent logs from eating all disk space!
default-address    → Custom IP ranges for container networks
dns                → Custom DNS servers for containers
live-restore       → Containers keep running if daemon restarts
userland-proxy     → false = better performance for port mapping
```

### After Changing Config

```bash
# Reload daemon configuration
sudo systemctl daemon-reload
sudo systemctl restart docker

# Verify config is applied
docker info | grep "Storage Driver"
docker info | grep "Logging Driver"
```

---

## 🧰 Essential Tools to Install Alongside Docker

```bash
# Docker Compose (if not installed via plugin)
# Usually included with docker-compose-plugin, verify:
docker compose version

# Dive — inspect Docker image layers
# (Excellent for optimizing image sizes!)
# Install: https://github.com/wagoodman/dive
wget https://github.com/wagoodman/dive/releases/download/v0.12.0/dive_0.12.0_linux_amd64.deb
sudo dpkg -i dive_0.12.0_linux_amd64.deb
dive nginx:latest   # Explore nginx image layers!

# lazydocker — Terminal UI for Docker
# (Best TUI for Docker management!)
curl https://raw.githubusercontent.com/jesseduffield/lazydocker/master/scripts/install_update_linux.sh | bash
lazydocker   # Beautiful terminal Docker dashboard

# ctop — Top-like interface for containers
sudo wget https://github.com/bcicen/ctop/releases/download/v0.7.7/ctop-0.7.7-linux-amd64 -O /usr/local/bin/ctop
sudo chmod +x /usr/local/bin/ctop
ctop   # Real-time container metrics
```

---

## 🧠 Memory Shortcuts for This Chapter

### Install Command: **"RAPI"**
```
R = Remove old versions
A = Add Docker repository + GPG key
P = Package install (docker-ce, containerd, compose)
I = Init (usermod, systemctl enable, start)
```

### Post-Install Checklist: **"GVES"**
```
G = Group (add user to docker group)
V = Version check (docker version)
E = Enable service (systemctl enable docker)
S = Smoke test (docker run hello-world)
```

---

## ❓ Quick Quiz

1. What is the difference between Docker Desktop and Docker Engine?
2. Why do we add user to the `docker` group?
3. What does `docker run hello-world` actually do behind the scenes?
4. Where is Docker's daemon config file?
5. What is `overlay2` and why is it the recommended storage driver?

<details>
<summary>Click for Answers</summary>

1. Docker Desktop = GUI + Engine + Compose + Kubernetes (for Windows/Mac). Docker Engine = CLI + Daemon only (for Linux servers).
2. To run docker commands without `sudo`. The docker group grants access to the Docker socket.
3. Checks for image locally → pulls from Docker Hub → creates container → runs the hello-world process → prints message → container exits.
4. `/etc/docker/daemon.json`
5. Overlay2 is a storage driver that efficiently layers filesystem changes. It uses copy-on-write and layer sharing to save disk space.

</details>

---

**← Previous: [02 - Architecture & Internals](./02-architecture-internals.md)** | **Next: [04 - Images Deep Dive](./04-images-deep-dive.md)** ➡️
