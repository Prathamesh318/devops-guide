# 🔤 Chapter 7: Variables & Facts

> **"Variables make your playbooks flexible. Facts make them smart."**

---

## 🤔 Why Do Variables Exist?

```
WITHOUT variables:
  - name: Install nginx on Ubuntu
    apt:
      name: nginx
      state: present
  
  → Hardcoded! Need a different version? Edit the playbook.
  → Different servers need different values? Create separate playbooks!

WITH variables:
  - name: Install {{ package_name }}
    apt:
      name: "{{ package_name }}"
      state: "{{ package_state }}"
  
  → Flexible! Change the variable, same playbook works everywhere.
  → Production uses latest, staging uses specific version.
```

---

## 📐 Variable Syntax (Jinja2)

```yaml
# Variables are referenced using double curly braces: {{ variable_name }}
# This is Jinja2 syntax (the templating engine Ansible uses)

# ⚠️ IMPORTANT YAML RULE:
# If a value STARTS with {{ }}, you MUST quote it!

# ✅ Correct:
  msg: "Hello {{ name }}"              # Mid-string = no quotes needed (but good practice)
  msg: "{{ name }}"                     # Starts with {{ = MUST quote!
  port: "{{ http_port }}"              # Starts with {{ = MUST quote!

# ❌ Wrong:
  msg: {{ name }}                       # Will cause YAML parse error!
  port: {{ http_port }}                 # Will break!
```

---

## 📍 Where to Define Variables (All 14 Places!)

### 1. In the Playbook (play vars)

```yaml
- name: My Play
  hosts: webservers
  vars:                              # ← Define here
    http_port: 80
    app_name: myapp
    packages:
    - nginx
    - python3
    - git

  tasks:
  - name: Install packages
    apt:
      name: "{{ packages }}"
      state: present
```

### 2. In a vars file (imported)

```yaml
# vars/web_config.yml
---
http_port: 80
https_port: 443
document_root: /var/www/html
max_clients: 100

# In playbook:
- name: Web setup
  hosts: webservers
  vars_files:
  - vars/web_config.yml              # ← Import the file

  tasks:
  - name: Show port
    debug:
      msg: "Port is {{ http_port }}"
```

### 3. In inventory (host/group vars)

```ini
# Inline in inventory
[webservers]
web1 ansible_host=192.168.1.11 http_port=8080   # ← Host variable

[webservers:vars]
document_root=/var/www/html            # ← Group variable
```

### 4. In group_vars/ directory

```yaml
# inventory/group_vars/webservers.yml  ← Auto-loaded for webservers group!
---
http_port: 80
nginx_worker_processes: 4
ssl_enabled: true

# inventory/group_vars/all.yml        ← Applies to ALL hosts
---
ntp_server: time.google.com
dns_server: 8.8.8.8
timezone: Asia/Kolkata
```

### 5. In host_vars/ directory

```yaml
# inventory/host_vars/web1.yml        ← Only for web1!
---
http_port: 8080                        # web1 uses custom port
ssl_certificate: /etc/ssl/web1.crt
is_primary: true
```

### 6. At the command line (highest priority!)

```bash
# Pass variables via command line
ansible-playbook site.yml -e "http_port=8080"
ansible-playbook site.yml --extra-vars "http_port=8080 app_env=production"
ansible-playbook site.yml -e "@vars/custom.yml"    # From a file
ansible-playbook site.yml -e '{"http_port": 8080, "debug": true}'  # JSON
```

### 7. Registered variables (from task output)

```yaml
- name: Check if nginx is installed
  command: which nginx
  register: nginx_check              # ← Save output to variable
  ignore_errors: true

- name: Show result
  debug:
    msg: "Nginx path: {{ nginx_check.stdout }}"
    # Available properties:
    # nginx_check.stdout       = standard output
    # nginx_check.stderr       = error output
    # nginx_check.rc           = return code (0 = success)
    # nginx_check.changed      = did the task change anything?
    # nginx_check.failed       = did the task fail?
```

### 8. set_fact (dynamic variables)

```yaml
- name: Calculate a value
  set_fact:
    max_memory: "{{ ansible_memory_mb.real.total // 2 }}MB"
    is_production: "{{ 'prod' in inventory_hostname }}"

- name: Show calculated value
  debug:
    msg: "Max memory: {{ max_memory }}, Production: {{ is_production }}"
```

---

## 📊 Variable Precedence (Most to Least)

```
Who wins when the SAME variable is defined in multiple places?

Priority (HIGHEST to LOWEST):
─────────────────────────────────────
 1. Extra vars (-e on command line)     ← ALWAYS WINS
 2. Task vars (set_fact, register)
 3. Block vars
 4. Role vars (vars/main.yml)
 5. Play vars_files
 6. Play vars
 7. Host facts (from setup module)
 8. host_vars/hostname.yml
 9. group_vars/groupname.yml
10. group_vars/all.yml
11. Inventory host vars
12. Inventory group vars
13. Role defaults (defaults/main.yml)   ← Lowest priority
─────────────────────────────────────

Memory trick:
  Command line > Task > Play > Host > Group > Role defaults
  "CTPHGR" → "Commands Take Priority, Hosts Get Roles"
```

### Why This Matters:

```
# Role default: http_port = 80
# group_vars:   http_port = 8080
# Command line: -e "http_port=9090"

# Result: http_port = 9090 (command line wins!)

# This lets you:
# 1. Set sensible defaults in roles
# 2. Override per-group in group_vars
# 3. Override per-host in host_vars
# 4. Force override from command line in emergencies
```

---

## 🧠 Facts — Auto-Discovered System Information

### What Are Facts?

```
Facts = Information that Ansible AUTOMATICALLY gathers from each server

When a playbook runs:
  TASK [Gathering Facts]    ← This is Ansible running the 'setup' module
  ok: [web1]                   It collects 100+ facts about each server

Facts include:
  ansible_hostname          = "web1"
  ansible_os_family         = "Debian"
  ansible_distribution      = "Ubuntu"
  ansible_distribution_version = "22.04"
  ansible_default_ipv4.address = "192.168.1.11"
  ansible_memory_mb.real.total = 4096
  ansible_processor_cores   = 2
  ansible_architecture      = "x86_64"
  ansible_devices            = {disk info}
  ansible_mounts             = [{mount points}]
```

### Using Facts in Playbooks

```yaml
- name: Use facts
  hosts: all
  tasks:
  - name: Show server info
    debug:
      msg: |
        Hostname: {{ ansible_hostname }}
        OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
        IP: {{ ansible_default_ipv4.address }}
        RAM: {{ ansible_memory_mb.real.total }} MB
        CPU Cores: {{ ansible_processor_cores }}
        Architecture: {{ ansible_architecture }}

  - name: Configure based on OS
    apt:
      name: nginx
      state: present
    when: ansible_os_family == "Debian"

  - name: Configure based on OS (RedHat)
    yum:
      name: nginx
      state: present
    when: ansible_os_family == "RedHat"

  - name: Set memory-based config
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    vars:
      max_memory: "{{ ansible_memory_mb.real.total // 2 }}"
```

### Disable Fact Gathering (Speed Up)

```yaml
- name: Quick play (no facts needed)
  hosts: all
  gather_facts: false             # ← Skip fact gathering = faster!
  
  tasks:
  - name: Just restart service
    service:
      name: nginx
      state: restarted
```

### Custom Facts

```bash
# On the managed node, create a fact file:
# /etc/ansible/facts.d/custom.fact

# INI format:
[app]
name=myapp
version=2.1.0
env=production

# Or JSON format:
# /etc/ansible/facts.d/custom.fact
{
  "app": {
    "name": "myapp",
    "version": "2.1.0"
  }
}
```

```yaml
# Access custom facts:
- name: Show custom fact
  debug:
    msg: "App: {{ ansible_local.custom.app.name }} v{{ ansible_local.custom.app.version }}"
```

---

## 🔧 Variable Data Types

```yaml
# String
app_name: "myapp"
greeting: Hello World

# Number
http_port: 80
max_retries: 3
pi: 3.14

# Boolean
debug_mode: true
ssl_enabled: false
is_production: yes        # yes/no also work in YAML

# List (Array)
packages:
- nginx
- python3
- git
# Or inline: packages: [nginx, python3, git]

# Dictionary (Map)
database:
  host: localhost
  port: 5432
  name: mydb
  user: admin
# Access: {{ database.host }} or {{ database['host'] }}

# Nested
app:
  server:
    name: web1
    port: 8080
    ssl:
      enabled: true
      cert: /etc/ssl/cert.pem
# Access: {{ app.server.ssl.cert }}
```

---

## 🔍 The debug Module — Your Best Friend

```yaml
# Print a variable's value
- debug:
    var: http_port                  # Shows: "http_port": 80

# Print a formatted message
- debug:
    msg: "Port is {{ http_port }}"  # Shows: "Port is 80"

# Print with verbosity level (only with -v flag)
- debug:
    msg: "Debug info: {{ some_var }}"
    verbosity: 2                    # Only shows with -vv or higher

# Dump ALL facts
- debug:
    var: ansible_facts
```

---

## 💡 Key Takeaways

```
1. Variables use {{ }} syntax (Jinja2)
2. Quote values that START with {{ }}
3. 14 places to define variables, each with different priority
4. Command line (-e) ALWAYS wins
5. Facts = auto-discovered system info (OS, IP, RAM, CPU)
6. group_vars/ and host_vars/ = cleanest way to organize variables
7. register = save task output as a variable
8. set_fact = dynamic variable creation
9. debug module = print variables for troubleshooting
```

---

**⬅️ Previous: [06 - Playbooks](./06-playbooks-core.md)** | **Next: [08 - Conditionals & Loops](./08-conditionals-loops.md)** ➡️
