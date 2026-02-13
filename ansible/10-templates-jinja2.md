# 📝 Chapter 10: Templates & Jinja2

> **"Templates turn static config files into dynamic, server-aware blueprints."**

---

## 🤔 Why Do Templates Exist?

```
Problem: Each server needs a SLIGHTLY different config file

Server 1 (nginx.conf):
  server_name web1.example.com;     ← Different hostname
  worker_processes 4;                ← Different cores

Server 2 (nginx.conf):
  server_name web2.example.com;     ← Different hostname
  worker_processes 8;                ← Different cores

Without templates:
  → Create separate config files for each server? 50 servers = 50 files!

With templates:
  → ONE template file with {{ variables }}
  → Ansible fills in the right values for each server automatically!
```

---

## 📐 Template Basics

### Template vs Copy

```yaml
# copy module → copies file AS IS (no variable substitution)
- name: Copy static file
  copy:
    src: files/index.html
    dest: /var/www/html/index.html

# template module → processes Jinja2 first, then copies result
- name: Deploy config from template
  template:
    src: templates/nginx.conf.j2            # ← .j2 = Jinja2 template
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
```

### Simple Template Example

```jinja
{# templates/nginx.conf.j2 #}
{# This is a Jinja2 comment — not included in final output #}

# Managed by Ansible — DO NOT EDIT MANUALLY
# Generated on {{ ansible_date_time.date }}

server {
    listen {{ nginx_port | default(80) }};
    server_name {{ ansible_hostname }}.{{ domain }};
    
    root {{ document_root }};
    index index.html;
    
    # Server info (auto-detected)
    # OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
    # IP: {{ ansible_default_ipv4.address }}
    # CPUs: {{ ansible_processor_cores }}
    
    worker_processes {{ ansible_processor_cores }};
}
```

**Result on web1 (4 cores, Ubuntu 22.04):**
```nginx
# Managed by Ansible — DO NOT EDIT MANUALLY
# Generated on 2026-02-12

server {
    listen 80;
    server_name web1.example.com;
    
    root /var/www/html;
    index index.html;
    
    # Server info (auto-detected)
    # OS: Ubuntu 22.04
    # IP: 192.168.1.11
    # CPUs: 4
    
    worker_processes 4;
}
```

---

## 🔧 Jinja2 Syntax Reference

### Variables

```jinja
{{ variable_name }}                    {# Output a variable #}
{{ server.hostname }}                  {# Access dict attribute #}
{{ servers[0] }}                       {# Access list item #}
{{ ansible_default_ipv4.address }}     {# Nested attribute #}
```

### Comments

```jinja
{# This is a Jinja2 comment — won't appear in output #}

{# Multi-line
   comment #}
```

### Conditionals in Templates

```jinja
{# templates/app.conf.j2 #}

{% if environment == "production" %}
DEBUG=false
LOG_LEVEL=warning
{% elif environment == "staging" %}
DEBUG=true
LOG_LEVEL=info
{% else %}
DEBUG=true
LOG_LEVEL=debug
{% endif %}

{% if ssl_enabled %}
listen 443 ssl;
ssl_certificate {{ ssl_cert_path }};
ssl_certificate_key {{ ssl_key_path }};
{% else %}
listen 80;
{% endif %}

{# Check if variable is defined #}
{% if custom_header is defined %}
add_header X-Custom "{{ custom_header }}";
{% endif %}
```

### Loops in Templates

```jinja
{# templates/hosts.j2 — Generate /etc/hosts #}
127.0.0.1   localhost
::1         localhost

# Managed hosts
{% for host in groups['all'] %}
{{ hostvars[host]['ansible_default_ipv4']['address'] }}   {{ host }}
{% endfor %}

{# Output:
127.0.0.1   localhost
::1         localhost

# Managed hosts
192.168.1.11   web1
192.168.1.12   web2
192.168.1.21   db1
#}
```

```jinja
{# templates/vhosts.conf.j2 — Multiple virtual hosts #}
{% for site in websites %}
server {
    listen {{ site.port | default(80) }};
    server_name {{ site.domain }};
    root {{ site.root }};
    
    {% if site.ssl | default(false) %}
    listen 443 ssl;
    ssl_certificate /etc/ssl/{{ site.domain }}.crt;
    {% endif %}
}

{% endfor %}
```

### Loop Controls

```jinja
{# Loop with index #}
{% for user in users %}
{{ loop.index }}. {{ user.name }} ({{ user.role }})
{% endfor %}

{# Loop variables:
   loop.index     = current iteration (1-based)
   loop.index0    = current iteration (0-based)
   loop.first     = True if first iteration
   loop.last      = True if last iteration
   loop.length    = total number of items
#}

{# Conditional inside loop #}
{% for package in packages %}
{% if package != "debug-tools" %}
- {{ package }}
{% endif %}
{% endfor %}
```

---

## 🔩 Filters — Transform Data

```jinja
{# Filters use the pipe (|) syntax #}

{# String filters #}
{{ hostname | upper }}                   → "WEB1"
{{ hostname | lower }}                   → "web1"
{{ hostname | capitalize }}              → "Web1"
{{ hostname | title }}                   → "Web1"
{{ "  spaces  " | trim }}               → "spaces"
{{ hostname | replace("web", "app") }}   → "app1"
{{ path | basename }}                    → "config.yml" (from /etc/app/config.yml)
{{ path | dirname }}                     → "/etc/app"

{# Default values #}
{{ http_port | default(80) }}            → 80 if http_port is not defined
{{ custom_var | default("none", true) }} → "none" if undefined or empty string

{# Number filters #}
{{ "42" | int }}                         → 42
{{ "3.14" | float }}                     → 3.14
{{ 1048576 | human_readable }}           → "1.0 MB"

{# List filters #}
{{ packages | join(", ") }}              → "nginx, python3, git"
{{ [3,1,2] | sort }}                     → [1, 2, 3]
{{ [1,2,2,3] | unique }}                 → [1, 2, 3]
{{ packages | length }}                  → 3
{{ users | map(attribute='name') | list }} → ["alice", "bob"]
{{ numbers | min }}                       → smallest number
{{ numbers | max }}                       → largest number
{{ [1,2,3] | first }}                    → 1
{{ [1,2,3] | last }}                     → 3
{{ list1 | union(list2) }}               → combined unique
{{ list1 | intersect(list2) }}           → common items
{{ list1 | difference(list2) }}          → items in list1 not in list2

{# Hash/Dict filters #}
{{ dict | to_json }}                     → JSON string
{{ dict | to_yaml }}                     → YAML string
{{ dict | to_nice_json }}                → Pretty JSON
{{ dict | to_nice_yaml }}                → Pretty YAML
{{ "password" | hash('sha512') }}        → SHA512 hash
{{ "secret" | b64encode }}               → base64 encoded
{{ "c2VjcmV0" | b64decode }}             → base64 decoded

{# IP address filters #}
{{ "192.168.1.0/24" | ipaddr('network') }}  → "192.168.1.0"
{{ ip_list | ipv4 }}                         → only IPv4 addresses

{# Type conversion #}
{{ "true" | bool }}                      → True
{{ value | string }}                     → convert to string
{{ value | type_debug }}                 → shows the variable type

{# Regex #}
{{ text | regex_search('pattern') }}     → find first match
{{ text | regex_replace('old', 'new') }} → replace pattern
```

---

## 🔍 Lookups — Pull Data from External Sources

```yaml
# Read from a file
- debug:
    msg: "{{ lookup('file', '/etc/hostname') }}"

# Read environment variable
- debug:
    msg: "Home is {{ lookup('env', 'HOME') }}"

# Generate password
- debug:
    msg: "{{ lookup('password', '/tmp/pass length=16 chars=ascii_letters,digits') }}"

# Read from URL
- debug:
    msg: "{{ lookup('url', 'https://api.ipify.org') }}"

# Read pipe output
- debug:
    msg: "{{ lookup('pipe', 'date +%Y-%m-%d') }}"

# Template lookup
- debug:
    msg: "{{ lookup('template', 'templates/greeting.j2') }}"
```

---

## 🎯 Real-World Template Examples

### PostgreSQL Config

```jinja
{# templates/postgresql.conf.j2 #}
# PostgreSQL Configuration — Managed by Ansible
# Auto-tuned for {{ ansible_memory_mb.real.total }}MB RAM

listen_addresses = '{{ pg_listen_addresses | default("localhost") }}'
port = {{ pg_port | default(5432) }}
max_connections = {{ pg_max_connections | default(100) }}

# Memory (auto-calculated from server RAM)
shared_buffers = {{ (ansible_memory_mb.real.total * 0.25) | int }}MB
effective_cache_size = {{ (ansible_memory_mb.real.total * 0.75) | int }}MB
work_mem = {{ (ansible_memory_mb.real.total * 0.25 / pg_max_connections | default(100)) | int }}MB

# Logging
log_directory = '{{ pg_log_dir | default("/var/log/postgresql") }}'
log_filename = 'postgresql-%Y-%m-%d.log'

{% if pg_replication_enabled | default(false) %}
# Replication
wal_level = replica
max_wal_senders = {{ pg_max_wal_senders | default(3) }}
{% endif %}
```

### Sudoers File

```jinja
{# templates/sudoers.j2 #}
# Sudoers — Managed by Ansible
# DO NOT EDIT MANUALLY

Defaults   env_reset
Defaults   secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

root ALL=(ALL:ALL) ALL

{% for user in sudo_users %}
{{ user.name }} ALL=(ALL) {% if user.nopasswd | default(false) %}NOPASSWD:{% endif %}ALL
{% endfor %}

{% for group in sudo_groups %}
%{{ group }} ALL=(ALL:ALL) ALL
{% endfor %}
```

---

## ✅ Template Validation

```yaml
# Validate config before deploying
- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: "nginx -t -c %s"            # ← Test config before placing!
  notify: Reload nginx

# If validation fails → file NOT placed → no broken service!
```

---

## 💡 Key Takeaways

```
1. Templates = config files with {{ variables }}
2. Jinja2 = the templating engine ({{ }}, {% %}, {# #})
3. template module = process .j2 file and copy result
4. Filters (|) = transform data (upper, default, join, etc.)
5. Lookups = pull data from files, env vars, URLs
6. validate = test config before deploying (prevents broken services!)
7. Always add "Managed by Ansible" comment to templates
8. .j2 extension = convention for Jinja2 templates
```

---

**⬅️ Previous: [09 - Roles & Galaxy](./09-roles-galaxy.md)** | **Next: [11 - Vault & Security](./11-vault-security.md)** ➡️
