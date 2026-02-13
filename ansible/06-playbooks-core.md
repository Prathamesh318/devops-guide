# 📜 Chapter 6: Playbooks — The Core of Ansible

> **"If ad-hoc commands are one-liners, playbooks are the full novel."**

---

## 🤔 Why Do Playbooks Exist?

```
Ad-hoc command:
  ansible webservers -m apt -a "name=nginx state=present" -b
  → Great for ONE task

But what if you need:
  1. Install nginx
  2. Copy config file
  3. Start nginx
  4. Open firewall port
  5. Create log directory
  → Running 5 ad-hoc commands manually? Messier than a shell script!

Playbook solves this:
  → All tasks in ONE file
  → Runs in ORDER
  → REUSABLE (run it again and again)
  → VERSION CONTROLLED (keep in Git)
  → DOCUMENTED (YAML is readable)
```

---

## 📐 Anatomy of a Playbook

```yaml
# site.yml — A complete playbook
---                                    # ← YAML document start
- name: Configure web servers          # ← PLAY 1 (name of this play)
  hosts: webservers                    # ← Which hosts to target
  become: true                         # ← Run as root (sudo)
  vars:                                # ← Variables for this play
    http_port: 80
    doc_root: /var/www/html

  tasks:                               # ← List of tasks to execute
  - name: Install nginx               # ← TASK 1 (descriptive name!)
    apt:                               # ← Module to use
      name: nginx                      # ← Module parameter
      state: present                   # ← Module parameter
      update_cache: yes

  - name: Start nginx                 # ← TASK 2
    service:
      name: nginx
      state: started
      enabled: yes

  - name: Copy index page             # ← TASK 3
    copy:
      content: "<h1>Hello from {{ inventory_hostname }}!</h1>"
      dest: "{{ doc_root }}/index.html"

  handlers:                            # ← Run ONLY when triggered
  - name: Restart nginx
    service:
      name: nginx
      state: restarted

- name: Configure database servers     # ← PLAY 2 (another play!)
  hosts: databases
  become: true

  tasks:
  - name: Install PostgreSQL
    apt:
      name: postgresql
      state: present
```

### Breaking It Down

```
┌─────────────────────────────────────────────────────────────┐
│  PLAYBOOK                                                    │
│  └── PLAY 1: "Configure web servers"                         │
│      ├── hosts: webservers                                   │
│      ├── vars: http_port=80                                  │
│      ├── TASK 1: "Install nginx"                             │
│      │   └── module: apt (name=nginx, state=present)        │
│      ├── TASK 2: "Start nginx"                               │
│      │   └── module: service (name=nginx, state=started)    │
│      ├── TASK 3: "Copy index page"                           │
│      │   └── module: copy (content=..., dest=...)           │
│      └── HANDLER: "Restart nginx"                            │
│          └── module: service (name=nginx, state=restarted)  │
│  └── PLAY 2: "Configure database servers"                    │
│      ├── hosts: databases                                    │
│      └── TASK 1: "Install PostgreSQL"                        │
│          └── module: apt (name=postgresql, state=present)   │
└─────────────────────────────────────────────────────────────┘

Key Terms:
  Playbook = The entire YAML file (can have multiple plays)
  Play     = A section targeting specific hosts
  Task     = A single action within a play
  Module   = The tool used to perform the action
  Handler  = A task that only runs when notified
```

---

## ▶️ Running Playbooks

```bash
# Basic run
ansible-playbook site.yml

# With inventory file
ansible-playbook -i inventory/production site.yml

# Dry run (check mode — don't make changes)
ansible-playbook site.yml --check

# Show changes (diff mode)
ansible-playbook site.yml --diff

# Verbose output
ansible-playbook site.yml -v     # More info
ansible-playbook site.yml -vvv   # Debug level

# Limit to specific hosts
ansible-playbook site.yml --limit web1
ansible-playbook site.yml --limit 'web1,web2'

# Start at a specific task
ansible-playbook site.yml --start-at-task "Copy index page"

# Step through tasks one by one (confirm each)
ansible-playbook site.yml --step

# List all tasks without running them
ansible-playbook site.yml --list-tasks

# List all hosts that would be affected
ansible-playbook site.yml --list-hosts

# Syntax check (verify YAML is valid)
ansible-playbook site.yml --syntax-check
```

---

## 🔔 Handlers — "Do This Only If Something Changed"

### Why Do Handlers Exist?

```
Problem:
  Task 1: Copy new nginx.conf       → config changed
  Task 2: Restart nginx             → always runs, even if config didn't change!
  
  Running playbook 10 times = 10 unnecessary restarts!

Solution — Handlers:
  Task 1: Copy new nginx.conf       → notify "restart nginx" if changed
  Handler: Restart nginx             → ONLY runs if notified!

  Run 1: Config changed  → handler runs    → nginx restarted ✅
  Run 2: Config same     → handler skipped → no restart 👍
```

### Handler Example

```yaml
---
- name: Configure nginx
  hosts: webservers
  become: true

  tasks:
  - name: Install nginx
    apt:
      name: nginx
      state: present

  - name: Copy nginx configuration
    copy:
      src: files/nginx.conf
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx              # ← Trigger handler IF this task changes

  - name: Copy site configuration
    copy:
      src: files/mysite.conf
      dest: /etc/nginx/sites-available/mysite.conf
    notify:                            # ← Can notify multiple handlers!
    - Restart nginx
    - Reload nginx

  handlers:
  - name: Restart nginx               # ← Only runs if notified
    service:
      name: nginx
      state: restarted

  - name: Reload nginx                # ← Another handler
    service:
      name: nginx
      state: reloaded
```

### Key Handler Rules:

```
1. Handlers run ONLY when notified by a task
2. Handlers run at the END of all tasks (not immediately)
3. Handlers run ONLY ONCE even if notified multiple times
4. Handler name MUST match the "notify" exactly
5. Handlers run in the ORDER they're defined, not notification order
```

---

## 📊 Playbook Execution Flow

```
┌────────────────────────────────────────────────────────────┐
│  ansible-playbook site.yml                                  │
│                                                             │
│  1. Parse YAML ─────────────────────────────────────────── │
│     └── Validate syntax, resolve variables                  │
│                                                             │
│  2. PLAY 1: "Configure web servers"                         │
│     │                                                       │
│     ├── Gather Facts (on all webservers) ──────────────── │
│     │   └── Collects OS, IP, RAM, disk info automatically  │
│     │                                                       │
│     ├── Task 1: "Install nginx" ─────────────────────────  │
│     │   └── Run on web1, web2, web3 (in parallel)          │
│     │   └── Results: changed / ok / failed                  │
│     │                                                       │
│     ├── Task 2: "Start nginx" ───────────────────────────  │
│     │   └── Run on web1, web2, web3 (in parallel)          │
│     │                                                       │
│     ├── Task 3: "Copy config" ───────────────────────────  │
│     │   └── Run on web1, web2, web3 (in parallel)          │
│     │   └── If changed → mark handler "restart nginx"      │
│     │                                                       │
│     └── HANDLERS (run at the end) ───────────────────────  │
│         └── "Restart nginx" (only if notified)              │
│                                                             │
│  3. PLAY 2: "Configure database servers"                    │
│     └── Same flow for databases group                       │
│                                                             │
│  4. PLAY RECAP ─────────────────────────────────────────── │
│     web1: ok=4  changed=2  unreachable=0  failed=0         │
│     web2: ok=4  changed=2  unreachable=0  failed=0         │
│     db1:  ok=2  changed=1  unreachable=0  failed=0         │
└────────────────────────────────────────────────────────────┘
```

---

## 📝 Real-World Playbook Examples

### Example 1: Complete Web Server Setup

```yaml
# webserver-setup.yml
---
- name: Setup Web Server from Scratch
  hosts: webservers
  become: true
  vars:
    domain: myapp.example.com
    doc_root: /var/www/{{ domain }}

  tasks:
  - name: Update apt cache
    apt:
      update_cache: yes
      cache_valid_time: 3600          # Don't update if updated < 1 hour ago

  - name: Install required packages
    apt:
      name:
      - nginx
      - python3
      - python3-pip
      - git
      - curl
      - ufw
      state: present

  - name: Create document root
    file:
      path: "{{ doc_root }}"
      state: directory
      owner: www-data
      group: www-data
      mode: '0755'

  - name: Deploy index.html
    copy:
      content: |
        <!DOCTYPE html>
        <html>
        <head><title>{{ domain }}</title></head>
        <body>
          <h1>Welcome to {{ domain }}!</h1>
          <p>Server: {{ inventory_hostname }}</p>
          <p>IP: {{ ansible_default_ipv4.address }}</p>
          <p>OS: {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
        </body>
        </html>
      dest: "{{ doc_root }}/index.html"
      owner: www-data
      group: www-data

  - name: Configure nginx site
    copy:
      content: |
        server {
            listen 80;
            server_name {{ domain }};
            root {{ doc_root }};
            index index.html;

            location / {
                try_files $uri $uri/ =404;
            }
        }
      dest: /etc/nginx/sites-available/{{ domain }}.conf
    notify: Reload nginx

  - name: Enable site
    file:
      src: /etc/nginx/sites-available/{{ domain }}.conf
      dest: /etc/nginx/sites-enabled/{{ domain }}.conf
      state: link
    notify: Reload nginx

  - name: Remove default site
    file:
      path: /etc/nginx/sites-enabled/default
      state: absent
    notify: Reload nginx

  - name: Configure UFW - Allow SSH
    ufw:
      rule: allow
      port: '22'

  - name: Configure UFW - Allow HTTP
    ufw:
      rule: allow
      port: '80'

  - name: Enable UFW
    ufw:
      state: enabled
      policy: deny

  - name: Ensure nginx is running
    service:
      name: nginx
      state: started
      enabled: yes

  handlers:
  - name: Reload nginx
    service:
      name: nginx
      state: reloaded
```

### Example 2: User Management

```yaml
# manage-users.yml
---
- name: Manage System Users
  hosts: all
  become: true

  vars:
    developers:
    - name: alice
      groups: developers,sudo
      shell: /bin/bash
    - name: bob
      groups: developers
      shell: /bin/bash
    - name: charlie
      groups: developers,docker
      shell: /bin/zsh

    removed_users:
    - olddev1
    - olddev2

  tasks:
  - name: Create developer group
    group:
      name: developers
      state: present

  - name: Create developer accounts
    user:
      name: "{{ item.name }}"
      groups: "{{ item.groups }}"
      shell: "{{ item.shell }}"
      create_home: yes
      state: present
    loop: "{{ developers }}"          # ← Loop over each developer

  - name: Set up SSH keys for developers
    authorized_key:
      user: "{{ item.name }}"
      key: "{{ lookup('file', 'keys/' + item.name + '.pub') }}"
    loop: "{{ developers }}"

  - name: Remove old users
    user:
      name: "{{ item }}"
      state: absent
      remove: yes                      # Also delete home directory
    loop: "{{ removed_users }}"
```

### Example 3: Application Deployment

```yaml
# deploy-app.yml
---
- name: Deploy Application
  hosts: app_servers
  become: true
  vars:
    app_name: myapp
    app_repo: https://github.com/company/myapp.git
    app_branch: main
    app_dir: /opt/{{ app_name }}
    app_user: deploy

  tasks:
  - name: Ensure app user exists
    user:
      name: "{{ app_user }}"
      system: yes
      shell: /bin/bash

  - name: Create app directory
    file:
      path: "{{ app_dir }}"
      state: directory
      owner: "{{ app_user }}"
      group: "{{ app_user }}"

  - name: Clone/update application code
    git:
      repo: "{{ app_repo }}"
      dest: "{{ app_dir }}"
      version: "{{ app_branch }}"
      force: yes
    become_user: "{{ app_user }}"
    notify: Restart application

  - name: Install Python dependencies
    pip:
      requirements: "{{ app_dir }}/requirements.txt"
      virtualenv: "{{ app_dir }}/venv"
    become_user: "{{ app_user }}"

  - name: Deploy systemd service file
    copy:
      content: |
        [Unit]
        Description={{ app_name }}
        After=network.target

        [Service]
        User={{ app_user }}
        WorkingDirectory={{ app_dir }}
        ExecStart={{ app_dir }}/venv/bin/python app.py
        Restart=always

        [Install]
        WantedBy=multi-user.target
      dest: /etc/systemd/system/{{ app_name }}.service
    notify:
    - Reload systemd
    - Restart application

  - name: Ensure app is running
    service:
      name: "{{ app_name }}"
      state: started
      enabled: yes

  handlers:
  - name: Reload systemd
    systemd:
      daemon_reload: yes

  - name: Restart application
    service:
      name: "{{ app_name }}"
      state: restarted
```

---

## 📊 Play Recap — Understanding Output

```
PLAY [Configure web servers] **********************************************

TASK [Gathering Facts] ****************************************************
ok: [web1]
ok: [web2]

TASK [Install nginx] ******************************************************
changed: [web1]               ← 🟡 Nginx was installed
ok: [web2]                    ← 🟢 Already installed

TASK [Start nginx] ********************************************************
ok: [web1]
ok: [web2]

TASK [Copy config] ********************************************************
changed: [web1]               ← 🟡 Config was changed → handler notified!
ok: [web2]                    ← 🟢 Config identical

RUNNING HANDLER [Reload nginx] ********************************************
changed: [web1]               ← Only web1 (because only web1's config changed)

PLAY RECAP ****************************************************************
web1  : ok=5    changed=3    unreachable=0    failed=0    skipped=0
web2  : ok=4    changed=0    unreachable=0    failed=0    skipped=0

Legend:
  ok       = task ran successfully, no change needed
  changed  = task made a change
  unreachable = couldn't connect via SSH
  failed   = task failed
  skipped  = task was skipped (condition not met)
```

---

## 🔧 Multi-Play Playbooks

```yaml
# site.yml — Master playbook with multiple plays
---
# Play 1: Common setup for ALL servers
- name: Common configuration
  hosts: all
  become: true
  tasks:
  - name: Update packages
    apt: update_cache=yes
  - name: Install common tools
    apt:
      name: [vim, curl, wget, htop, tree]
      state: present
  - name: Set timezone
    timezone:
      name: Asia/Kolkata

# Play 2: Web-specific setup
- name: Configure web servers
  hosts: webservers
  become: true
  tasks:
  - name: Install nginx
    apt: name=nginx state=present
  - name: Start nginx
    service: name=nginx state=started enabled=yes

# Play 3: Database-specific setup
- name: Configure database servers
  hosts: databases
  become: true
  tasks:
  - name: Install PostgreSQL
    apt: name=postgresql state=present
  - name: Start PostgreSQL
    service: name=postgresql state=started enabled=yes
```

---

## 📎 Including & Importing Playbooks

```yaml
# site.yml — Master playbook that includes others
---
- import_playbook: common.yml        # Run common setup first
- import_playbook: webservers.yml    # Then web servers
- import_playbook: databases.yml     # Then databases
- import_playbook: monitoring.yml    # Then monitoring
```

**import vs include:**
```
import_playbook:
  ✅ Parsed at playbook LOAD time (static)
  ✅ Shows in --list-tasks
  ✅ Can't use loops

include_playbook: (deprecated, use import)

import_tasks: (for tasks within a play)
  ✅ Static, parsed at load time

include_tasks: (for tasks within a play)
  ✅ Dynamic, parsed at RUNTIME
  ✅ Can use loops and conditionals
```

---

## 💡 Key Takeaways

```
1. Playbook = YAML file with one or more plays
2. Play = targets specific hosts with specific tasks
3. Tasks run IN ORDER, top to bottom
4. Handlers run ONLY when notified, at END of play
5. Use --check for dry runs, --diff for changes
6. --syntax-check before running in production!
7. Name every play and task descriptively
8. Use import_playbook to organize large automations
```

---

**⬅️ Previous: [05 - Ad-hoc Commands](./05-ad-hoc-commands.md)** | **Next: [07 - Variables & Facts](./07-variables-facts.md)** ➡️
