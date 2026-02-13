# ⚡ Chapter 5: Ad-hoc Commands

> **"Sometimes you need to do ONE thing on 100 servers. That's where ad-hoc commands shine."**

---

## 🤔 What Are Ad-hoc Commands?

```
Ad-hoc = "For this specific purpose, right now"

Playbook = A recipe that you save and reuse (like a cookbook)
Ad-hoc   = A quick command you run once (like ordering takeout)

Use ad-hoc when:
  ✅ Quick one-time task
  ✅ Checking something across servers
  ✅ Emergency fix (restart a service NOW)
  ✅ Gathering information

Use playbooks when:
  ✅ Repeatable process
  ✅ Multiple steps
  ✅ Complex logic
  ✅ Needs to be version controlled
```

---

## 📐 Anatomy of an Ad-hoc Command

```
ansible  <hosts>  -m <module>  -a "<arguments>"  [options]
  │         │        │             │                  │
  │         │        │             │                  └─ Extra flags
  │         │        │             └─ Module parameters
  │         │        └─ Which module to use
  │         └─ Which hosts to target
  └─ The ansible command

Example:
ansible webservers -m apt -a "name=nginx state=present" -b
        └────────┘  └───┘   └──────────────────────────┘ └┘
          WHERE     WHAT         WITH WHAT PARAMS        AS ROOT
```

---

## 🧰 Essential Modules for Ad-hoc Commands

### 1. ping — Test Connectivity

```bash
# NOT a network ping! Tests Ansible connectivity + Python
ansible all -m ping

# Output:
# web1 | SUCCESS => { "ping": "pong" }
# web2 | SUCCESS => { "ping": "pong" }
# db1  | UNREACHABLE! => { "msg": "SSH connection failed" }

# Ping specific group
ansible webservers -m ping

# Ping with verbose output
ansible all -m ping -v       # Some detail
ansible all -m ping -vvv     # Full SSH debug
```

### 2. shell / command — Run Commands

```bash
# command module (default, safer — no shell features)
ansible all -m command -a "uptime"
ansible all -m command -a "whoami"
ansible all -m command -a "hostname"

# shell module (supports pipes, redirects, env vars)
ansible all -m shell -a "df -h | head -5"
ansible all -m shell -a "cat /etc/os-release | grep PRETTY"
ansible all -m shell -a "ps aux | grep nginx | wc -l"
ansible all -m shell -a "echo $HOME"

# Difference:
# command → no shell features (|, >, $, &&) — SAFER
# shell   → full shell features — more powerful but riskier
```

### 3. apt / yum — Package Management

```bash
# Install a package (Ubuntu/Debian)
ansible webservers -m apt -a "name=nginx state=present" -b

# Install a package (CentOS/RHEL)
ansible webservers -m yum -a "name=httpd state=present" -b

# Install multiple packages
ansible all -m apt -a "name=git,curl,wget state=present" -b

# Remove a package
ansible webservers -m apt -a "name=nginx state=absent" -b

# Update all packages
ansible all -m apt -a "upgrade=yes update_cache=yes" -b

# Install specific version
ansible all -m apt -a "name=nginx=1.18.0-0ubuntu1 state=present" -b

# state values:
#   present  → install if not installed
#   absent   → remove if installed
#   latest   → install/upgrade to latest version
```

### 4. service / systemd — Service Management

```bash
# Start a service
ansible webservers -m service -a "name=nginx state=started" -b

# Stop a service
ansible webservers -m service -a "name=nginx state=stopped" -b

# Restart a service
ansible webservers -m service -a "name=nginx state=restarted" -b

# Enable service at boot + start it
ansible webservers -m service -a "name=nginx state=started enabled=yes" -b

# Check service status
ansible webservers -m shell -a "systemctl status nginx" -b
```

### 5. copy — Copy Files to Servers

```bash
# Copy a file to all servers
ansible all -m copy -a "src=./my-config.conf dest=/etc/myapp/config.conf" -b

# Copy with permissions
ansible all -m copy -a "src=./script.sh dest=/usr/local/bin/script.sh mode=0755 owner=root" -b

# Create a file with content (no src file needed!)
ansible all -m copy -a "content='Hello from Ansible!\n' dest=/tmp/hello.txt"

# Backup before overwriting
ansible all -m copy -a "src=./nginx.conf dest=/etc/nginx/nginx.conf backup=yes" -b
```

### 6. file — Manage Files & Directories

```bash
# Create a directory
ansible all -m file -a "path=/opt/myapp state=directory mode=0755" -b

# Create an empty file
ansible all -m file -a "path=/tmp/test.log state=touch"

# Change file permissions
ansible all -m file -a "path=/var/log/myapp.log mode=0644 owner=www-data" -b

# Delete a file
ansible all -m file -a "path=/tmp/test.log state=absent"

# Create a symbolic link
ansible all -m file -a "src=/opt/myapp/current dest=/var/www/html state=link" -b
```

### 7. user / group — User Management

```bash
# Create a user
ansible all -m user -a "name=deploy shell=/bin/bash state=present" -b

# Create user with specific UID and groups
ansible all -m user -a "name=appuser uid=1050 groups=www-data,docker shell=/bin/bash" -b

# Remove a user
ansible all -m user -a "name=olduser state=absent remove=yes" -b

# Create a group
ansible all -m group -a "name=developers state=present" -b
```

### 8. setup — Gather System Facts

```bash
# Gather ALL facts
ansible web1 -m setup

# Filter facts (these are super useful!)
ansible all -m setup -a "filter=ansible_distribution"
ansible all -m setup -a "filter=ansible_os_family"
ansible all -m setup -a "filter=ansible_default_ipv4"
ansible all -m setup -a "filter=ansible_hostname"
ansible all -m setup -a "filter=ansible_memory_mb"
ansible all -m setup -a "filter=ansible_processor_cores"
ansible all -m setup -a "filter=ansible_disk*"
```

### 9. lineinfile — Edit a Single Line in File

```bash
# Add a line to a file
ansible all -m lineinfile -a "path=/etc/hosts line='192.168.1.100 app.local'" -b

# Replace a line matching a pattern
ansible all -m lineinfile -a "path=/etc/ssh/sshd_config regexp='^PermitRootLogin' line='PermitRootLogin no'" -b

# Remove a line
ansible all -m lineinfile -a "path=/etc/hosts regexp='old-server' state=absent" -b
```

### 10. git — Clone Repositories

```bash
# Clone a repository
ansible webservers -m git -a "repo=https://github.com/user/app.git dest=/var/www/myapp version=main" -b

# Update to latest
ansible webservers -m git -a "repo=https://github.com/user/app.git dest=/var/www/myapp version=main update=yes" -b
```

---

## 🎛️ Useful Flags & Options

```bash
# -b (--become): Run as root (sudo)
ansible all -m apt -a "name=nginx state=present" -b

# -K (--ask-become-pass): Ask for sudo password
ansible all -m shell -a "whoami" -b -K

# -k (--ask-pass): Ask for SSH password
ansible all -m ping -k

# -u (--user): Specify SSH user
ansible all -m ping -u deploy

# -i (--inventory): Specify inventory file
ansible all -m ping -i my-inventory

# -v, -vv, -vvv, -vvvv: Verbosity levels
ansible all -m ping -v      # Some detail
ansible all -m ping -vvv    # Full debug

# --limit: Restrict to specific hosts
ansible all -m ping --limit web1
ansible all -m ping --limit 'web1,web2'

# --check: Dry run (don't actually do anything)
ansible all -m apt -a "name=nginx state=present" -b --check

# --diff: Show file changes
ansible all -m copy -a "content='new content' dest=/tmp/file.txt" --diff

# -f (--forks): Parallel connections
ansible all -m ping -f 20    # Run on 20 hosts simultaneously

# -o (--one-line): Compact output
ansible all -m ping -o
```

---

## 🎯 Common Scenarios (Copy & Use!)

### Scenario 1: Check servers health
```bash
# Uptime
ansible all -m shell -a "uptime" -o

# Disk usage
ansible all -m shell -a "df -h /"

# Memory usage
ansible all -m shell -a "free -m"

# CPU load
ansible all -m shell -a "cat /proc/loadavg"

# Running processes count
ansible all -m shell -a "ps aux | wc -l"
```

### Scenario 2: Emergency operations
```bash
# Restart a crashed service across all servers
ansible webservers -m service -a "name=nginx state=restarted" -b

# Kill a rogue process
ansible all -m shell -a "pkill -f 'rogue-process'" -b

# Check if a port is open
ansible all -m shell -a "ss -tlnp | grep 80"
```

### Scenario 3: Quick security audit
```bash
# Check who's logged in
ansible all -m shell -a "who"

# Check failed login attempts
ansible all -m shell -a "grep 'Failed password' /var/log/auth.log | tail -5" -b

# Check sudo users
ansible all -m shell -a "grep -v '^#' /etc/sudoers | grep -v '^$'" -b

# Check open ports
ansible all -m shell -a "ss -tlnp" -b
```

### Scenario 4: Bulk user management
```bash
# Create deploy user on all servers
ansible all -m user -a "name=deploy shell=/bin/bash groups=sudo" -b

# Set up SSH key for deploy user
ansible all -m authorized_key -a "user=deploy key='{{ lookup(\"file\", \"~/.ssh/id_ed25519.pub\") }}'" -b
```

---

## 📊 Understanding Output

```
ansible webservers -m apt -a "name=nginx state=present" -b

# Output colors:
# 🟢 GREEN  = ok (no change needed, already in desired state)
# 🟡 YELLOW = changed (Ansible made a change)
# 🔴 RED    = failed (something went wrong)
# ⚫ DARK   = unreachable (can't connect via SSH)

# Example output:
web1 | CHANGED => {                    ← Yellow: nginx was installed
    "changed": true,
    "msg": "nginx installed"
}
web2 | SUCCESS => {                    ← Green: already installed
    "changed": false,
    "msg": "nginx already present"
}
web3 | FAILED! => {                    ← Red: something broke
    "changed": false,
    "msg": "Could not find package nginx"
}
db1 | UNREACHABLE! => {                ← Dark: can't SSH
    "msg": "Failed to connect to host via SSH"
}
```

---

## ⚠️ Command vs Shell vs Raw

```
┌───────────────────────────────────────────────────────────────┐
│  Module    │ Shell Features │ Idempotent │ When to Use         │
│───────────────────────────────────────────────────────────────│
│  command   │ ❌ No pipes,  │ ❌ No      │ Simple commands     │
│            │    redirects   │            │ (safest option)     │
│───────────────────────────────────────────────────────────────│
│  shell     │ ✅ Full shell │ ❌ No      │ Need pipes, env     │
│            │    features    │            │ vars, redirects     │
│───────────────────────────────────────────────────────────────│
│  raw       │ ✅ Full shell │ ❌ No      │ No Python on target │
│            │    (no Python) │            │ (bootstrap only)    │
└───────────────────────────────────────────────────────────────┘

Golden Rule: Use MODULES instead of shell commands whenever possible!
  ❌ ansible all -m shell -a "apt-get install nginx" -b
  ✅ ansible all -m apt -a "name=nginx state=present" -b
```

---

## 💡 Key Takeaways

```
1. Ad-hoc = quick, one-time commands across multiple servers
2. Format: ansible <hosts> -m <module> -a "<args>"
3. Use MODULES over shell commands (idempotent!)
4. -b = become root (sudo)
5. ping module tests Ansible connectivity, NOT network ping
6. color coding: green=ok, yellow=changed, red=failed
7. --check = dry run, --diff = show changes
8. Ad-hoc for quick tasks, Playbooks for repeatable automation
```

---

**⬅️ Previous: [04 - Inventory Deep Dive](./04-inventory-deep-dive.md)** | **Next: [06 - Playbooks — The Core](./06-playbooks-core.md)** ➡️
