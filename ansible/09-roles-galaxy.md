# 📦 Chapter 9: Roles & Ansible Galaxy

> **"Roles are how professionals organize Ansible code. No production playbook should live without them."**

---

## 🤔 Why Do Roles Exist?

```
WITHOUT Roles (Monolithic Playbook):
───────────────────────────────────────
site.yml → 500 lines long!
  → Install nginx
  → Configure nginx
  → Install PostgreSQL
  → Configure PostgreSQL
  → Create users
  → Configure firewall
  → Setup monitoring
  → Deploy application
  → ... everything in ONE file!

Problems:
  ❌ Unreadable (500+ lines of YAML)
  ❌ Not reusable (can't share "just the nginx part")
  ❌ Hard to test (can't test nginx setup independently)
  ❌ Multiple people can't work on same file easily
  ❌ No separation of concerns

WITH Roles:
───────────────────────────────────────
site.yml → 15 lines!
  roles:
  - common
  - nginx
  - postgresql
  - app_deploy
  - monitoring

Each role is:
  ✅ Self-contained (own tasks, vars, templates, files)
  ✅ Reusable (use nginx role in all projects!)
  ✅ Testable (test nginx role independently)
  ✅ Shareable (publish to Ansible Galaxy)
  ✅ Organized (everything has its place)
```

---

## 📐 Role Directory Structure

```
roles/
└── nginx/                         ← Role name
    ├── defaults/
    │   └── main.yml               ← Default variables (lowest priority)
    ├── vars/
    │   └── main.yml               ← Role variables (high priority)
    ├── tasks/
    │   └── main.yml               ← Main task list (ENTRY POINT)
    ├── handlers/
    │   └── main.yml               ← Handlers
    ├── templates/
    │   └── nginx.conf.j2          ← Jinja2 templates
    ├── files/
    │   └── index.html             ← Static files
    ├── meta/
    │   └── main.yml               ← Role metadata & dependencies
    └── README.md                  ← Documentation
```

### What Goes Where?

```
defaults/main.yml  → Variables that users SHOULD override
                     (e.g., http_port: 80, max_clients: 100)

vars/main.yml      → Variables that users SHOULDN'T override
                     (e.g., nginx_config_dir: /etc/nginx)

tasks/main.yml     → The actual work (install, configure, start)
                     This is the ENTRY POINT of the role

handlers/main.yml  → Event-driven tasks (restart nginx when config changes)

templates/         → Jinja2 template files (.j2 extension)
                     Dynamic config files with {{ variables }}

files/             → Static files to copy (no variable substitution)
                     HTML files, scripts, certificates

meta/main.yml      → Dependencies and Galaxy metadata
                     "This role requires the 'common' role first"
```

---

## 🔨 Creating a Role: Step by Step

### Method 1: ansible-galaxy init (Scaffolding)

```bash
# Creates the full directory structure automatically
ansible-galaxy init roles/nginx

# Creates:
# roles/nginx/
# ├── defaults/main.yml
# ├── files/
# ├── handlers/main.yml
# ├── meta/main.yml
# ├── README.md
# ├── tasks/main.yml
# ├── templates/
# ├── tests/
# │   ├── inventory
# │   └── test.yml
# └── vars/main.yml
```

### Method 2: Manual (Only Create What You Need)

```bash
mkdir -p roles/nginx/{tasks,handlers,templates,defaults}
```

### Complete Nginx Role Example

```yaml
# roles/nginx/defaults/main.yml
# ─── Default variables (users can override) ───
---
nginx_port: 80
nginx_server_name: localhost
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_document_root: /var/www/html
nginx_index: index.html
nginx_packages:
- nginx

# roles/nginx/vars/main.yml
# ─── Internal variables (don't override) ───
---
nginx_config_dir: /etc/nginx
nginx_sites_available: /etc/nginx/sites-available
nginx_sites_enabled: /etc/nginx/sites-enabled
nginx_service_name: nginx
```

```yaml
# roles/nginx/tasks/main.yml
# ─── Main task list ───
---
- name: Install nginx packages
  apt:
    name: "{{ nginx_packages }}"
    state: present
    update_cache: yes

- name: Create document root
  file:
    path: "{{ nginx_document_root }}"
    state: directory
    owner: www-data
    group: www-data
    mode: '0755'

- name: Deploy nginx main config
  template:
    src: nginx.conf.j2
    dest: "{{ nginx_config_dir }}/nginx.conf"
    owner: root
    group: root
    mode: '0644'
  notify: Restart nginx

- name: Deploy default site config
  template:
    src: default-site.conf.j2
    dest: "{{ nginx_sites_available }}/default"
  notify: Reload nginx

- name: Enable default site
  file:
    src: "{{ nginx_sites_available }}/default"
    dest: "{{ nginx_sites_enabled }}/default"
    state: link
  notify: Reload nginx

- name: Deploy index page
  copy:
    src: index.html
    dest: "{{ nginx_document_root }}/index.html"
    owner: www-data
    group: www-data

- name: Ensure nginx is running and enabled
  service:
    name: "{{ nginx_service_name }}"
    state: started
    enabled: yes
```

```yaml
# roles/nginx/handlers/main.yml
---
- name: Restart nginx
  service:
    name: "{{ nginx_service_name }}"
    state: restarted

- name: Reload nginx
  service:
    name: "{{ nginx_service_name }}"
    state: reloaded
```

```
# roles/nginx/templates/nginx.conf.j2
# ─── Jinja2 template ───
user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

```
# roles/nginx/templates/default-site.conf.j2
server {
    listen {{ nginx_port }};
    server_name {{ nginx_server_name }};
    root {{ nginx_document_root }};
    index {{ nginx_index }};

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```html
<!-- roles/nginx/files/index.html -->
<!DOCTYPE html>
<html>
<head><title>Welcome</title></head>
<body>
    <h1>Server is running! ✅</h1>
    <p>Configured by Ansible</p>
</body>
</html>
```

```yaml
# roles/nginx/meta/main.yml
---
galaxy_info:
  author: your_name
  description: Install and configure Nginx
  min_ansible_version: "2.9"
  platforms:
  - name: Ubuntu
    versions: [focal, jammy]
  - name: Debian
    versions: [bullseye, bookworm]
  galaxy_tags:
  - nginx
  - web
  - server

dependencies:
- role: common                    # ← This role runs BEFORE nginx!
```

---

## ▶️ Using Roles in Playbooks

### Method 1: roles keyword (Classic)

```yaml
# site.yml
---
- name: Configure web servers
  hosts: webservers
  become: true

  roles:
  - common                        # ← Uses default variables
  - nginx                         # ← Uses default variables
  - role: nginx                   # ← With custom variables
    vars:
      nginx_port: 8080
      nginx_server_name: mysite.com
```

### Method 2: include_role (Dynamic)

```yaml
- name: Configure servers
  hosts: all
  tasks:
  - name: Setup nginx on web servers
    include_role:
      name: nginx
    when: "'webservers' in group_names"

  - name: Setup postgres on db servers
    include_role:
      name: postgresql
    when: "'databases' in group_names"
```

### Method 3: import_role (Static)

```yaml
- name: Configure servers
  hosts: webservers
  tasks:
  - name: Pre-setup tasks
    debug:
      msg: "Starting setup..."

  - import_role:
      name: common

  - import_role:
      name: nginx
    vars:
      nginx_port: 8080
```

---

## 🌐 Ansible Galaxy — The Role Marketplace

### What Is Galaxy?

```
Ansible Galaxy = npm for Ansible
  → Community-shared roles
  → Don't reinvent the wheel!
  → Need nginx? Someone already wrote a role!
  → Install, customize, use

Website: https://galaxy.ansible.com
```

### Using Galaxy

```bash
# Search for roles
ansible-galaxy search nginx
ansible-galaxy search postgresql --platforms Ubuntu

# Get info about a role
ansible-galaxy info geerlingguy.nginx

# Install a role
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install geerlingguy.postgresql

# Install to specific directory
ansible-galaxy install geerlingguy.nginx -p ./roles/

# Install specific version
ansible-galaxy install geerlingguy.nginx,3.1.0

# List installed roles
ansible-galaxy list

# Remove a role
ansible-galaxy remove geerlingguy.nginx
```

### requirements.yml — Pin Role Versions

```yaml
# requirements.yml — Like package.json for roles!
---
roles:
- name: geerlingguy.nginx
  version: "3.1.0"

- name: geerlingguy.postgresql
  version: "3.4.0"

- name: geerlingguy.docker
  version: "6.1.0"

# Install from Git
- name: custom_role
  src: https://github.com/company/ansible-role-custom.git
  version: v1.0.0

collections:
- name: amazon.aws
  version: "5.0.0"
- name: community.general
  version: "6.0.0"
```

```bash
# Install all roles from requirements
ansible-galaxy install -r requirements.yml

# Force reinstall
ansible-galaxy install -r requirements.yml --force
```

---

## 🏗️ Collections — The New Way

```
Collections = Bundles of roles, modules, plugins, and docs

Role = just tasks/handlers/templates
Collection = roles + modules + plugins + everything!

Galaxy used to be ONLY roles.
Now Galaxy hosts COLLECTIONS too.
```

```bash
# Install a collection
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install community.general

# Use modules from a collection
- name: Create EC2 instance
  amazon.aws.ec2_instance:        # ← Collection.module format
    instance_type: t2.micro
    image_id: ami-12345678
```

---

## 📂 Best Practice: Project Layout with Roles

```
production-ansible/
├── ansible.cfg
├── requirements.yml               ← External role dependencies
├── inventory/
│   ├── production/
│   │   ├── hosts
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   ├── webservers.yml
│   │   │   └── databases.yml
│   │   └── host_vars/
│   │       └── web1.yml
│   └── staging/
│       ├── hosts
│       └── group_vars/
├── playbooks/
│   ├── site.yml                   ← Master playbook
│   ├── webservers.yml
│   └── databases.yml
├── roles/
│   ├── common/                    ← YOUR roles
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── defaults/main.yml
│   ├── nginx/
│   ├── postgresql/
│   └── app_deploy/
└── galaxy_roles/                   ← Downloaded roles
    ├── geerlingguy.nginx/
    └── geerlingguy.docker/
```

```ini
# ansible.cfg
[defaults]
roles_path = ./roles:./galaxy_roles    # ← Look in both directories
```

---

## 💡 Key Takeaways

```
1. Roles = organized, reusable, shareable units of automation
2. Structure: tasks/ handlers/ templates/ files/ defaults/ vars/ meta/
3. defaults/ = variables users SHOULD override (lowest priority)
4. vars/ = variables users SHOULDN'T override
5. tasks/main.yml = entry point (runs automatically)
6. meta/main.yml = dependencies (run other roles first)
7. Galaxy = community marketplace for roles and collections
8. requirements.yml = pin role/collection versions
9. Collections = bundles of roles + modules + plugins
10. Always use roles for production playbooks!
```

---

**⬅️ Previous: [08 - Conditionals & Loops](./08-conditionals-loops.md)** | **Next: [10 - Templates & Jinja2](./10-templates-jinja2.md)** ➡️
