# 🏗️ Chapter 2: Architecture — How Ansible Actually Works

> **"Once you understand HOW Ansible works internally, everything else makes sense."**

---

## 🎯 The Big Picture

```
┌────────────────────────────────────────────────────────────────────┐
│                    ANSIBLE ARCHITECTURE                             │
│                                                                     │
│  ┌──────────────────────────────────────┐                          │
│  │         CONTROL NODE (Your PC)       │                          │
│  │                                       │                          │
│  │  ┌───────────┐  ┌──────────────┐     │                          │
│  │  │ Playbook  │  │  Inventory   │     │                          │
│  │  │ (YAML)    │  │ (host list)  │     │                          │
│  │  └─────┬─────┘  └──────┬───────┘     │                          │
│  │        │               │              │                          │
│  │        ▼               ▼              │                          │
│  │  ┌────────────────────────────┐      │                          │
│  │  │      ANSIBLE ENGINE        │      │                          │
│  │  │                            │      │                          │
│  │  │  ┌────────┐ ┌──────────┐  │      │                          │
│  │  │  │Modules │ │ Plugins  │  │      │                          │
│  │  │  └────────┘ └──────────┘  │      │                          │
│  │  └────────────┬───────────────┘      │                          │
│  │               │                       │                          │
│  └───────────────┼───────────────────────┘                          │
│                  │                                                   │
│                  │ SSH (port 22)                                     │
│                  │ No agent needed!                                  │
│                  │                                                   │
│     ┌────────────┼─────────────────────────────┐                    │
│     │            ▼            ▼            ▼    │                    │
│     │     ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│     │     │ Server 1 │ │ Server 2 │ │ Server 3 │                   │
│     │     │ (Python) │ │ (Python) │ │ (Python) │                   │
│     │     └──────────┘ └──────────┘ └──────────┘                   │
│     │           MANAGED NODES                    │                   │
│     └────────────────────────────────────────────┘                   │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🧩 Component Deep Dive

### 1. Control Node — The Brain

```
┌─────────────────────────────────────────────────────────────┐
│  CONTROL NODE = Where Ansible is installed                   │
│                                                              │
│  Can be:                                                     │
│  • Your laptop/desktop                                       │
│  • A dedicated automation server                             │
│  • A CI/CD server (Jenkins, GitLab CI)                       │
│  • NOT a Windows machine (Ansible can't run on Windows)      │
│    └─ But you can use WSL (Windows Subsystem for Linux) ✅   │
│                                                              │
│  What lives here:                                            │
│  📄 Playbooks     — Your automation recipes                  │
│  📋 Inventory     — List of servers to manage                │
│  ⚙️ ansible.cfg   — Configuration settings                   │
│  🔑 SSH Keys      — For authentication                       │
│  📦 Roles         — Reusable automation packages             │
└─────────────────────────────────────────────────────────────┘
```

**Why does the Control Node exist this way?**
- Everything is centralized — one place to manage all automation
- Version control friendly — all files are text (can use Git)
- No database needed — everything is files on disk
- Easy to move — copy files to a new machine, done!

### 2. Managed Nodes — The Targets

```
┌─────────────────────────────────────────────────────────────┐
│  MANAGED NODE = Servers that Ansible configures              │
│                                                              │
│  Requirements (ONLY TWO!):                                   │
│  1. SSH access   — Ansible connects via SSH                  │
│  2. Python 3     — Ansible modules run Python on the node    │
│                                                              │
│  That's it. No agent. No special software.*                  │
│                                                              │
│  * Most Linux distros come with both out of the box!         │
│                                                              │
│  Can be:                                                     │
│  🐧 Linux servers (most common)                              │
│  🪟 Windows servers (uses WinRM instead of SSH)              │
│  🌐 Network devices (routers, switches)                      │
│  ☁️  Cloud resources (AWS, Azure, GCP)                       │
│  🐳 Docker containers                                        │
└─────────────────────────────────────────────────────────────┘
```

### 3. Inventory — The Phone Book

```
"Ansible, WHO should you run this on?"

Static Inventory (simple text file):
────────────────────────────────────
[webservers]             ← Group name
web1.example.com         ← Server 1
web2.example.com         ← Server 2
192.168.1.10             ← Can use IP too

[databases]
db1.example.com
db2.example.com

[production:children]    ← Group of groups!
webservers
databases

Dynamic Inventory (auto-generated):
────────────────────────────────────
→ Script/plugin that queries AWS/Azure/GCP
→ Automatically finds all your cloud servers
→ No manual updating needed!
```

**Why does Inventory exist?**
- Ansible needs to know WHICH servers to manage
- Groups let you target specific server types
- Variables per host/group make servers unique
- Without it, you'd specify IPs every single time

### 4. Modules — The Workers

```
┌─────────────────────────────────────────────────────────────┐
│  MODULE = A unit of work that Ansible can perform            │
│                                                              │
│  Think of modules as TOOLS in a toolbox:                     │
│                                                              │
│  🔧 apt/yum     → Install packages                          │
│  📄 copy        → Copy files to servers                      │
│  📝 template    → Generate files from templates              │
│  ⚙️ service     → Start/stop services                        │
│  👤 user        → Create/manage users                        │
│  📂 file        → Manage files and directories               │
│  🔗 git         → Clone repositories                         │
│  📦 pip         → Install Python packages                    │
│  🐳 docker      → Manage Docker containers                   │
│  ☁️ aws         → Manage AWS resources                       │
│  💬 shell       → Run any shell command                      │
│  🔥 firewalld   → Manage firewall rules                      │
│                                                              │
│  Ansible has 3,000+ built-in modules!                        │
└─────────────────────────────────────────────────────────────┘
```

**Why do Modules exist instead of just running shell commands?**

```
Shell command (BAD approach):
  - shell: apt-get install nginx
  Problems:
    ❌ Not idempotent (runs every time even if installed)
    ❌ OS-specific (only works on Debian/Ubuntu)
    ❌ No error checking
    ❌ No change reporting

Module (GOOD approach):
  - apt:
      name: nginx
      state: present
  Benefits:
    ✅ Idempotent (skips if already installed)
    ✅ Cross-platform (use 'package' for any OS)
    ✅ Built-in error handling
    ✅ Reports: changed / ok / failed
```

### 5. Plugins — The Enhancers

```
┌─────────────────────────────────────────────────────────────┐
│  PLUGINS = Extend Ansible's core functionality               │
│                                                              │
│  Types:                                                      │
│  • Connection plugins  → HOW to connect (SSH, WinRM, Docker)│
│  • Callback plugins    → Output formatting                   │
│  • Lookup plugins      → Pull data from external sources     │
│  • Filter plugins      → Transform data in templates         │
│  • Inventory plugins   → Dynamic inventory sources           │
│                                                              │
│  Modules do the WORK on managed nodes.                       │
│  Plugins extend HOW Ansible itself functions.                │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔄 How Ansible Executes — Step by Step

When you run a playbook, here's EXACTLY what happens internally:

```
Step 1: READ
────────────
Ansible reads your playbook (YAML) and inventory

Step 2: PARSE
────────────
Ansible parses tasks, resolves variables, checks syntax

Step 3: CONNECT
────────────
Ansible opens SSH connections to target hosts
(Uses connection pooling — doesn't reconnect for each task!)

Step 4: GATHER FACTS
────────────
Ansible runs the 'setup' module on each host
Collects: OS, IP, RAM, CPU, disk, hostname, etc.
(This is why first task always says "Gathering Facts")

Step 5: GENERATE MODULE CODE
────────────
For each task, Ansible:
  1. Takes the module's Python code
  2. Injects your parameters (name: nginx, state: present)
  3. Creates a temporary Python script

Step 6: TRANSFER
────────────
Ansible copies the generated Python script to the remote host
Location: ~/.ansible/tmp/ on the remote server

Step 7: EXECUTE
────────────
Ansible runs the Python script on the remote host
The module does its job and returns JSON result

Step 8: REPORT
────────────
Ansible receives the JSON result:
{
  "changed": true,     ← Something was modified
  "msg": "nginx installed"
}

Step 9: CLEANUP
────────────
Ansible deletes the temporary script from the remote host

Step 10: NEXT
────────────
Repeat steps 5-9 for the next task
```

### Visual Flow:

```
Your PC (Control Node)                    Remote Server
┌──────────────────────┐                  ┌──────────────────┐
│                      │                  │                  │
│ 1. Read playbook     │                  │                  │
│ 2. Parse tasks       │                  │                  │
│                      │   SSH Connect    │                  │
│ 3. Open connection ──┼──────────────────┤                  │
│                      │                  │                  │
│ 4. "Gather facts!"  ─┼──── setup.py ───►│ Run setup.py     │
│    ◄ facts JSON  ────┼──────────────────┤ Return facts     │
│                      │                  │                  │
│ 5. Generate module   │                  │                  │
│    (apt.py + args)   │                  │                  │
│                      │   Copy module    │                  │
│ 6. Transfer ─────────┼─────────────────►│ /tmp/ansible_xxx │
│                      │                  │                  │
│ 7. "Execute!" ───────┼─────────────────►│ python apt.py    │
│    ◄ result JSON ────┼──────────────────┤ {changed: true}  │
│                      │                  │                  │
│ 8. Report result     │                  │ 9. Cleanup /tmp  │
│    (changed/ok/fail) │                  │                  │
│                      │                  │                  │
│ 10. Next task...     │                  │                  │
└──────────────────────┘                  └──────────────────┘
```

---

## 🧠 Key Concept: Idempotency in Action

```
First Run:
  TASK [Install nginx]
  changed: [server1]      ← Nginx was NOT installed, so Ansible installed it
  changed: [server2]      ← Same

Second Run (SAME playbook):
  TASK [Install nginx]
  ok: [server1]           ← Nginx is ALREADY installed, so Ansible skipped it
  ok: [server2]           ← Same

Third Run:
  TASK [Install nginx]
  ok: [server1]           ← Still installed, still skipped
  ok: [server2]           ← Same

PLAY RECAP:
  First run:  changed=5    ok=0    ← Made 5 changes
  Second run: changed=0    ok=5    ← No changes needed
  Third run:  changed=0    ok=5    ← Still no changes
```

**This is the magic of idempotency:**
- Run 1: Ansible DOES the work
- Run 2-1000: Ansible VERIFIES the work is done, does nothing
- Safe to run anytime, anywhere, as many times as you want

---

## 📂 Ansible File Structure

When you start working with Ansible, here's how your project looks:

```
my-ansible-project/
├── ansible.cfg              ← Configuration (where to find things)
├── inventory/
│   ├── production           ← Production server list
│   ├── staging              ← Staging server list
│   └── group_vars/
│       ├── all.yml           ← Variables for ALL servers
│       ├── webservers.yml    ← Variables for web servers
│       └── databases.yml     ← Variables for DB servers
├── playbooks/
│   ├── site.yml             ← Master playbook
│   ├── webserver.yml        ← Web server playbook
│   └── database.yml         ← Database playbook
├── roles/
│   ├── nginx/               ← Nginx role
│   ├── postgresql/          ← PostgreSQL role
│   └── common/              ← Common setup role
└── files/
    ├── nginx.conf           ← Configuration files
    └── ssh_keys/            ← SSH keys
```

**Why this structure?**
- **Separation of concerns** — inventory is separate from playbooks
- **Environment isolation** — production ≠ staging
- **Reusability** — roles can be shared across projects
- **Version control** — everything is a file, perfect for Git

---

## 🔑 Authentication — How Ansible Logs In

```
Method 1: SSH Key-based (Recommended ✅)
─────────────────────────────────────────
Control Node has private key → Managed Node has public key
No password needed. Secure. Automated.

  ~/.ssh/id_rsa (private) ──SSH──► ~/.ssh/authorized_keys (public)

Method 2: Password-based (Not recommended ❌)
─────────────────────────────────────────
ansible all -m ping -k     # -k = ask for SSH password
Problem: Can't automate if you have to type password!

Method 3: Become (Privilege Escalation)
─────────────────────────────────────────
Ansible connects as regular user → uses sudo for admin tasks

  ansible all -m apt -a "name=nginx state=present" -b
                                                     └─ -b = become (sudo)
```

---

## 🔀 Parallelism — How Ansible Handles Multiple Servers

```
Default: Ansible runs on 5 hosts at a time (configurable)

Inventory: [webservers]
  web1, web2, web3, web4, web5, web6, web7, web8, web9, web10

Task: Install nginx

Batch 1 (parallel):  web1, web2, web3, web4, web5  ──► Install nginx
                      ↓ All done? Next batch
Batch 2 (parallel):  web6, web7, web8, web9, web10 ──► Install nginx
                      ↓ All done? Next task

This is controlled by 'forks' in ansible.cfg:
  [defaults]
  forks = 10     ← Run on 10 hosts simultaneously
```

**Why not run on ALL hosts at once?**
- SSH connections use memory
- Too many simultaneous connections can overwhelm the control node
- Some tasks need sequential execution (database migrations)
- `serial: 2` in playbooks = rolling updates (2 at a time)

---

## 📊 Execution Strategies

```
Strategy 1: LINEAR (default)
────────────────────────────
  Task 1: Install nginx   → Run on ALL hosts → Wait for all → Next task
  Task 2: Start nginx     → Run on ALL hosts → Wait for all → Next task
  Task 3: Copy config     → Run on ALL hosts → Wait for all → Done

  ✅ Predictable, easy to debug
  ❌ Slow if one host is laggy

Strategy 2: FREE
────────────────────────────
  Each host runs through ALL tasks independently
  host1: Task 1 → Task 2 → Task 3 (done!)
  host2: Task 1 → Task 2 (still running...)
  host3: Task 1 → Task 2 → Task 3 (done!)

  ✅ Faster overall
  ❌ Harder to debug, output mixed together
```

---

## 💡 Key Takeaways

```
1. Control Node = Your machine (where Ansible lives)
2. Managed Nodes = Target servers (only need SSH + Python)
3. Modules = Workers that do specific tasks (3000+ available)
4. Plugins = Extend Ansible's core capabilities
5. SSH = The transport layer (no agents!)
6. Execution = Generate Python → Transfer → Execute → Report → Cleanup
7. Idempotent = Safe to run repeatedly
8. Parallel = Runs on multiple hosts simultaneously (forks)
```

---

**⬅️ Previous: [01 - Introduction](./01-introduction-why-ansible.md)** | **Next: [03 - Installation & Setup](./03-installation-setup.md)** ➡️
