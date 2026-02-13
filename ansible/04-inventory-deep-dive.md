# 📋 Chapter 4: Inventory Deep Dive

> **"If you don't tell Ansible WHO to manage, it can't manage anything."**

---

## 🧠 Why Does Inventory Exist?

```
Without Inventory:
  ansible 192.168.1.1,192.168.1.2,192.168.1.3 -m ping
  └── Typing IPs every time? Unmanageable at scale!

With Inventory:
  ansible webservers -m ping
  └── Clean, grouped, documented, reusable!

Inventory answers THREE questions:
  1. WHAT servers exist?              → Hosts
  2. HOW are they organized?          → Groups
  3. WHAT makes each unique?          → Variables
```

---

## 📄 Static Inventory (INI Format)

### Basic Format

```ini
# inventory/hosts
# This is the simplest inventory file

# Ungrouped hosts (belong to 'all' and 'ungrouped' groups)
server1.example.com
192.168.1.100

# Group: webservers
[webservers]
web1 ansible_host=192.168.1.11 ansible_user=deploy
web2 ansible_host=192.168.1.12 ansible_user=deploy
web3 ansible_host=192.168.1.13 ansible_user=deploy

# Group: databases
[databases]
db1 ansible_host=192.168.1.21 ansible_user=dbadmin
db2 ansible_host=192.168.1.22 ansible_user=dbadmin

# Group: load_balancers
[load_balancers]
lb1 ansible_host=192.168.1.5 ansible_user=root

# Meta-group: production (combines groups)
[production:children]
webservers
databases
load_balancers

# Variables for ALL hosts in webservers group
[webservers:vars]
http_port=80
max_connections=1000

# Variables for ALL hosts
[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### Understanding Each Part

```
[webservers]                          ← GROUP NAME (square brackets)
web1                                  ← HOST ALIAS (friendly name)
  ansible_host=192.168.1.11           ← Real IP/hostname
  ansible_user=deploy                 ← SSH username for this host

[production:children]                 ← META-GROUP (group of groups)
webservers                            ← Child group 1
databases                             ← Child group 2

[webservers:vars]                     ← GROUP VARIABLES
http_port=80                          ← Variable: applies to all webservers
```

---

## 📄 Static Inventory (YAML Format)

```yaml
# inventory/hosts.yml
# Same inventory, but in YAML format (more readable)

all:
  vars:
    ansible_python_interpreter: /usr/bin/python3
  
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.1.11
          ansible_user: deploy
        web2:
          ansible_host: 192.168.1.12
          ansible_user: deploy
        web3:
          ansible_host: 192.168.1.13
          ansible_user: deploy
      vars:
        http_port: 80
        max_connections: 1000
    
    databases:
      hosts:
        db1:
          ansible_host: 192.168.1.21
          ansible_user: dbadmin
        db2:
          ansible_host: 192.168.1.22
          ansible_user: dbadmin
    
    load_balancers:
      hosts:
        lb1:
          ansible_host: 192.168.1.5
          ansible_user: root
    
    production:
      children:
        webservers:
        databases:
        load_balancers:
```

---

## 📦 Special Inventory Variables

```yaml
# Connection variables — HOW Ansible connects
ansible_host: 192.168.1.10           # The actual IP/FQDN
ansible_port: 22                      # SSH port (default: 22)
ansible_user: deploy                  # SSH username
ansible_ssh_private_key_file: ~/.ssh/key  # SSH private key
ansible_connection: ssh               # Connection type (ssh, local, winrm)

# Privilege escalation — Running as root
ansible_become: true                  # Enable sudo
ansible_become_method: sudo           # How to become (sudo, su, pbrun)
ansible_become_user: root             # Which user to become
ansible_become_password: "{{ vault_sudo_pass }}"  # Sudo password

# Python — Where Python is on the target
ansible_python_interpreter: /usr/bin/python3

# Windows-specific
ansible_connection: winrm
ansible_winrm_server_cert_validation: ignore
```

---

## 🗂️ Inventory Directory Structure

For larger projects, split inventory into multiple files:

```
inventory/
├── production/
│   ├── hosts                 ← Host definitions
│   ├── group_vars/
│   │   ├── all.yml           ← Variables for ALL production hosts
│   │   ├── webservers.yml    ← Variables for webservers group
│   │   └── databases.yml     ← Variables for databases group
│   └── host_vars/
│       ├── web1.yml          ← Variables specific to web1
│       └── db1.yml           ← Variables specific to db1
├── staging/
│   ├── hosts
│   ├── group_vars/
│   │   └── all.yml
│   └── host_vars/
└── development/
    ├── hosts
    └── group_vars/
        └── all.yml
```

**Why this structure?**
```
Problem: All variables in one file = messy, hard to manage
Solution: 
  → group_vars/webservers.yml = variables ONLY for web servers
  → host_vars/web1.yml = variables ONLY for web1
  → Different environments = different directories

Usage:
  ansible-playbook -i inventory/production site.yml    # Production
  ansible-playbook -i inventory/staging site.yml       # Staging
```

### Example group_vars and host_vars

```yaml
# inventory/production/group_vars/webservers.yml
---
http_port: 80
https_port: 443
max_connections: 5000
document_root: /var/www/html
nginx_worker_processes: auto

# inventory/production/group_vars/databases.yml
---
db_port: 5432
max_connections: 200
shared_buffers: 256MB
backup_schedule: "0 2 * * *"

# inventory/production/host_vars/web1.yml
---
server_role: primary
ssl_certificate: /etc/ssl/web1.crt
custom_vhost: web1.example.com
```

---

## 🎯 Host Patterns — Targeting Specific Hosts

```bash
# Target ALL hosts
ansible all -m ping

# Target a specific group
ansible webservers -m ping

# Target a specific host
ansible web1 -m ping

# Target multiple groups (OR - union)
ansible 'webservers:databases' -m ping

# Target intersection (AND - must be in BOTH groups)
ansible 'webservers:&production' -m ping

# Target exclusion (NOT - exclude a group)
ansible 'all:!databases' -m ping

# Combine patterns
ansible 'production:!databases:&webservers' -m ping

# Wildcard matching
ansible 'web*' -m ping          # web1, web2, web3...
ansible '*.example.com' -m ping

# Regex matching
ansible '~web[0-9]+' -m ping    # web1, web2, web10...

# Index-based (first, last, specific)
ansible 'webservers[0]' -m ping     # First host
ansible 'webservers[-1]' -m ping    # Last host
ansible 'webservers[0:2]' -m ping   # First 3 hosts
```

---

## 🔮 Dynamic Inventory

### Why Dynamic Inventory?

```
Problem with Static Inventory:
  → You add 50 new EC2 instances in AWS
  → Now you have to MANUALLY add all 50 to your inventory file
  → Instances are terminated → you remove them manually
  → New instances created → add them again
  → This doesn't scale!

Dynamic Inventory solves this:
  → Ansible asks AWS: "What instances do you have?"
  → AWS returns the list automatically
  → Inventory is always up-to-date
  → No manual maintenance!
```

### AWS EC2 Dynamic Inventory

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2

filters:
  tag:Environment:
    - production
  instance-state-name:
    - running

keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.region
    prefix: region
  - key: instance_type
    separator: ""

compose:
  ansible_host: public_ip_address
```

```bash
# Install AWS collection
ansible-galaxy collection install amazon.aws

# Test dynamic inventory
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph
```

---

## 🔍 Useful Inventory Commands

```bash
# List all hosts in inventory
ansible all --list-hosts

# List hosts in a specific group
ansible webservers --list-hosts

# Show inventory as a graph (tree view)
ansible-inventory --graph

# Sample output:
# @all:
#   |--@ungrouped:
#   |--@webservers:
#   |  |--web1
#   |  |--web2
#   |  |--web3
#   |--@databases:
#   |  |--db1
#   |  |--db2
#   |--@production:
#   |  |--@webservers:
#   |  |--@databases:

# Show all variables for a host
ansible-inventory --host web1

# Show entire inventory as JSON
ansible-inventory --list

# Verify inventory file syntax
ansible-inventory -i inventory/hosts --list
```

---

## 🧮 Ranges & Shortcuts

```ini
# Instead of listing each host:
[webservers]
web1 ansible_host=192.168.1.11
web2 ansible_host=192.168.1.12
web3 ansible_host=192.168.1.13
web4 ansible_host=192.168.1.14
web5 ansible_host=192.168.1.15

# Use ranges!
[webservers]
web[1:5] ansible_host=192.168.1.1[1:5]

# Letter ranges also work
[servers]
server-[a:f].example.com    # server-a through server-f

# With step
[webservers]
web[01:20:2]                 # web01, web03, web05... web19 (step of 2)
```

---

## 🔒 Default Groups (Always Exist)

```
Every inventory has TWO automatic groups:

1. "all"        → Contains EVERY host in the inventory
2. "ungrouped"  → Contains hosts NOT in any custom group

hierarchy:
  all
  ├── ungrouped
  │   └── random-server.com
  ├── webservers
  │   ├── web1
  │   └── web2
  └── databases
      └── db1
```

---

## 💡 Key Takeaways

```
1. Inventory = Who Ansible manages (hosts + groups + variables)
2. Two formats: INI (simple) and YAML (structured)
3. group_vars/ and host_vars/ = variables per group/host
4. Host patterns = flexible targeting (wildcards, regex, unions)
5. Dynamic inventory = auto-discover cloud servers
6. Ranges = shorthand for numbered hosts (web[1:10])
7. "all" and "ungrouped" = built-in groups
8. Always keep separate inventories for prod/staging/dev
```

---

**⬅️ Previous: [03 - Installation & Setup](./03-installation-setup.md)** | **Next: [05 - Ad-hoc Commands](./05-ad-hoc-commands.md)** ➡️
