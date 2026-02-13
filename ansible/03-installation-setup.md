# ⚙️ Chapter 3: Installation & Setup

> **"5 minutes from zero to your first Ansible command."**

---

## 📋 Requirements

```
┌─────────────────────────────────────────────────────────────┐
│  Control Node (WHERE Ansible is installed):                  │
│  • Linux or macOS (NOT Windows natively)                     │
│  • Python 3.9+ installed                                     │
│  • pip (Python package manager)                              │
│                                                              │
│  ⚠️ Windows users: Use WSL (Windows Subsystem for Linux)    │
│     Ansible CANNOT be installed directly on Windows          │
│     But it CAN MANAGE Windows systems!                       │
│                                                              │
│  Managed Nodes (WHAT Ansible manages):                       │
│  • SSH access                                                │
│  • Python 3 installed (most Linux distros have this)         │
│  • That's it!                                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 🪟 For Windows Users: Setting Up WSL

Since you're on Windows, here's your setup path:

### Step 1: Install WSL

```powershell
# Open PowerShell as Administrator
wsl --install

# This installs Ubuntu by default
# Restart your computer when prompted
```

### Step 2: First-time WSL Setup

```bash
# After reboot, Ubuntu will open and ask for:
# 1. Username (e.g., your name)
# 2. Password (for sudo)

# Update packages
sudo apt update && sudo apt upgrade -y
```

### Step 3: Install Ansible in WSL

```bash
# Method 1: Using apt (Ubuntu/Debian) — Recommended
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

# Verify installation
ansible --version

# Expected output:
# ansible [core 2.16.x]
#   config file = /etc/ansible/ansible.cfg
#   configured module search path = ['/home/user/.ansible/plugins/modules']
#   ansible python module location = /usr/lib/python3/dist-packages/ansible
#   python version = 3.x.x
```

---

## 🐧 For Linux Users

### Ubuntu/Debian

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

### CentOS/RHEL/Fedora

```bash
# CentOS/RHEL 8+
sudo dnf install -y ansible-core

# Or using pip
sudo dnf install -y python3-pip
pip3 install ansible
```

### Using pip (Any Linux — Universal Method)

```bash
# Install pip if not already installed
sudo apt install -y python3-pip    # Debian/Ubuntu
# OR
sudo dnf install -y python3-pip   # CentOS/RHEL

# Install Ansible via pip
pip3 install ansible

# Add to PATH if needed
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 🍎 For macOS Users

```bash
# Using Homebrew (recommended)
brew install ansible

# Or using pip
pip3 install ansible
```

---

## ✅ Verify Installation

```bash
# Check Ansible version
ansible --version

# Check all Ansible tools installed
which ansible
which ansible-playbook
which ansible-galaxy
which ansible-vault
which ansible-doc

# Quick test (ping localhost)
ansible localhost -m ping

# Expected output:
# localhost | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

---

## 🏠 Setting Up Your Practice Environment

### Option 1: Using Vagrant (Virtual Machines) — Recommended for Learning

```
Why Vagrant?
  → Creates disposable VMs on your machine
  → Break something? Destroy and recreate in seconds
  → No cloud costs
  → Closest to real servers
```

```bash
# Install VirtualBox + Vagrant
# Download from:
#   VirtualBox: https://www.virtualbox.org/
#   Vagrant: https://www.vagrantup.com/

# Create a practice directory
mkdir ~/ansible-practice && cd ~/ansible-practice

# Create Vagrantfile (defines your VMs)
cat > Vagrantfile << 'EOF'
Vagrant.configure("2") do |config|
  
  # Control Node (where Ansible runs)
  config.vm.define "control" do |control|
    control.vm.box = "ubuntu/jammy64"
    control.vm.hostname = "control"
    control.vm.network "private_network", ip: "192.168.56.10"
    control.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
  end
  
  # Managed Node 1 (web server)
  config.vm.define "web1" do |web1|
    web1.vm.box = "ubuntu/jammy64"
    web1.vm.hostname = "web1"
    web1.vm.network "private_network", ip: "192.168.56.11"
    web1.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
      vb.cpus = 1
    end
  end
  
  # Managed Node 2 (web server)
  config.vm.define "web2" do |web2|
    web2.vm.box = "ubuntu/jammy64"
    web2.vm.hostname = "web2"
    web2.vm.network "private_network", ip: "192.168.56.12"
    web2.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
      vb.cpus = 1
    end
  end
  
  # Managed Node 3 (database server)
  config.vm.define "db1" do |db1|
    db1.vm.box = "ubuntu/jammy64"
    db1.vm.hostname = "db1"
    db1.vm.network "private_network", ip: "192.168.56.13"
    db1.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
      vb.cpus = 1
    end
  end
  
end
EOF

# Start all VMs
vagrant up

# SSH into control node
vagrant ssh control

# Inside control node, install Ansible
sudo apt update && sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

### Option 2: Using Docker Containers (Lightweight)

```bash
# Create a docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3'
services:
  control:
    image: ubuntu:22.04
    hostname: control
    command: sleep infinity
    networks:
      ansible_net:
        ipv4_address: 172.20.0.10

  web1:
    image: ubuntu:22.04
    hostname: web1
    command: sleep infinity
    networks:
      ansible_net:
        ipv4_address: 172.20.0.11

  web2:
    image: ubuntu:22.04
    hostname: web2
    command: sleep infinity
    networks:
      ansible_net:
        ipv4_address: 172.20.0.12

  db1:
    image: ubuntu:22.04
    hostname: db1
    command: sleep infinity
    networks:
      ansible_net:
        ipv4_address: 172.20.0.13

networks:
  ansible_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
EOF

docker-compose up -d
docker exec -it control bash
# Install Ansible inside the control container
apt update && apt install -y ansible openssh-client sshpass
```

### Option 3: Using Cloud (AWS/Azure Free Tier)

```
Create 3-4 small VMs:
  → 1 Control node (t2.micro)
  → 2-3 Managed nodes (t2.micro)
  → All in same VPC/subnet
  → Security group: Allow SSH (port 22)
```

---

## 🔑 Setting Up SSH Access

Ansible needs SSH access to managed nodes. Here's how to set it up:

### Generate SSH Key Pair (on Control Node)

```bash
# Generate SSH key (press Enter for all prompts)
ssh-keygen -t ed25519 -C "ansible-control"

# This creates:
# ~/.ssh/id_ed25519       ← Private key (KEEP SECRET!)
# ~/.ssh/id_ed25519.pub   ← Public key (copy to servers)
```

### Copy Public Key to Managed Nodes

```bash
# Copy to each managed node
ssh-copy-id user@192.168.56.11    # web1
ssh-copy-id user@192.168.56.12    # web2
ssh-copy-id user@192.168.56.13    # db1

# Test SSH without password
ssh user@192.168.56.11    # Should connect without asking password!
exit
```

### For Vagrant (Already Done)

```bash
# Vagrant automatically sets up SSH keys!
# You can SSH using:
vagrant ssh web1

# Or get the SSH config:
vagrant ssh-config web1
# This shows the private key location and port
```

---

## 📋 Creating Your First Inventory

```bash
# Create project directory
mkdir -p ~/ansible-practice && cd ~/ansible-practice

# Create inventory file
cat > inventory << 'EOF'
# Ansible Inventory File
# Format: hostname ansible_host=IP ansible_user=username

[webservers]
web1 ansible_host=192.168.56.11 ansible_user=vagrant
web2 ansible_host=192.168.56.12 ansible_user=vagrant

[databases]
db1 ansible_host=192.168.56.13 ansible_user=vagrant

[all:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

**Understanding the Inventory:**
```
[webservers]                          ← Group name
web1                                  ← Alias (friendly name)
  ansible_host=192.168.56.11          ← Actual IP/hostname
  ansible_user=vagrant                ← SSH username

[all:vars]                            ← Variables for ALL hosts
ansible_python_interpreter=/usr/bin/python3  ← Tell Ansible where Python is
```

---

## ⚙️ Creating ansible.cfg

```bash
# Create ansible.cfg in your project directory
cat > ansible.cfg << 'EOF'
[defaults]
# Path to inventory file
inventory = ./inventory

# Default SSH user
remote_user = vagrant

# Don't check host keys (for lab only!)
host_key_checking = False

# Number of parallel connections
forks = 10

# Default become method
become = True
become_method = sudo
become_user = root

# Reduce output noise
deprecation_warnings = False
interpreter_python = auto_silent

# Retry files location
retry_files_enabled = False

# Colored output
force_color = True
EOF
```

**Why does ansible.cfg exist?**
```
Without ansible.cfg:
  ansible -i /path/to/inventory -u vagrant --ask-become-pass webservers -m ping
  └── too many flags every time!

With ansible.cfg:
  ansible webservers -m ping
  └── everything is pre-configured!

Ansible looks for ansible.cfg in this order:
  1. ANSIBLE_CONFIG environment variable
  2. ./ansible.cfg (current directory) ← Most common
  3. ~/.ansible.cfg (home directory)
  4. /etc/ansible/ansible.cfg (global)
```

---

## 🧪 Your First Ansible Commands!

### Test 1: Ping all servers

```bash
# Ping all hosts in inventory
ansible all -m ping

# Expected output:
# web1 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
# web2 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
# db1 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

### Test 2: Ping specific groups

```bash
# Ping only web servers
ansible webservers -m ping

# Ping only databases
ansible databases -m ping
```

### Test 3: Run a command on all servers

```bash
# Check uptime on all servers
ansible all -m shell -a "uptime"

# Check disk space
ansible all -m shell -a "df -h"

# Check OS version
ansible all -m shell -a "cat /etc/os-release"

# Check memory
ansible all -m shell -a "free -m"
```

### Test 4: Gather facts from servers

```bash
# Gather and display ALL facts from web1
ansible web1 -m setup

# Filter specific facts
ansible web1 -m setup -a "filter=ansible_os_family"
ansible web1 -m setup -a "filter=ansible_distribution*"
ansible web1 -m setup -a "filter=ansible_memory_mb"
```

---

## 📂 Your Final Project Structure

```
~/ansible-practice/
├── ansible.cfg          ← Configuration (created above)
├── inventory            ← Server list (created above)
├── Vagrantfile          ← VM definitions (if using Vagrant)
└── (playbooks will go here in next chapters!)
```

---

## 🧹 Troubleshooting Common Setup Issues

### Issue 1: "Permission denied"
```bash
# Problem: SSH key not copied to managed node
# Solution:
ssh-copy-id user@target-server

# Or for Vagrant:
ansible all -m ping --private-key=~/.vagrant.d/insecure_private_key
```

### Issue 2: "HOST KEY VERIFICATION FAILED"
```bash
# Problem: New server, SSH doesn't trust it yet
# Solution: Add to ansible.cfg
[defaults]
host_key_checking = False

# Or: Accept the key manually first
ssh user@target-server   # Type 'yes' when prompted
```

### Issue 3: "MODULE FAILURE" / Python not found
```bash
# Problem: Python not installed on managed node
# Solution: Install Python on the target
ssh user@target-server
sudo apt install -y python3

# Or in ansible.cfg:
ansible_python_interpreter=/usr/bin/python3
```

### Issue 4: "sudo: a password is required"
```bash
# Problem: Target user needs sudo password
# Solution 1: Use --ask-become-pass flag
ansible all -m ping -b --ask-become-pass

# Solution 2: Configure passwordless sudo on target
# On the managed node:
echo "vagrant ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/vagrant
```

---

## 💡 Key Takeaways

```
1. Ansible runs on Linux/macOS (Windows users use WSL)
2. Install with: apt, dnf, pip, or brew
3. Managed nodes only need: SSH + Python 3
4. Inventory file = list of servers and groups
5. ansible.cfg = pre-configure defaults
6. SSH keys = passwordless authentication
7. First command: ansible all -m ping
8. Practice environment: Vagrant, Docker, or Cloud VMs
```

---

**⬅️ Previous: [02 - Architecture](./02-architecture-how-it-works.md)** | **Next: [04 - Inventory Deep Dive](./04-inventory-deep-dive.md)** ➡️
