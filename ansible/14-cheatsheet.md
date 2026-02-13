# 📋 Chapter 14: Ansible Cheat Sheet

> **Your ultimate quick reference guide!**

---

## ⚡ Essential Commands

```bash
# ─── Ad-hoc Commands ───
ansible all -m ping                          # Test connectivity
ansible all -m shell -a "uptime"             # Run command
ansible all -m apt -a "name=nginx state=present" -b  # Install package
ansible all -m service -a "name=nginx state=started" -b  # Start service
ansible all -m copy -a "src=f.txt dest=/tmp/" # Copy file
ansible all -m setup                         # Gather facts
ansible all -m setup -a "filter=ansible_os*" # Filter facts

# ─── Playbook Commands ───
ansible-playbook site.yml                    # Run playbook
ansible-playbook site.yml --check            # Dry run
ansible-playbook site.yml --diff             # Show changes
ansible-playbook site.yml -v / -vvv          # Verbose
ansible-playbook site.yml --limit web1       # Limit hosts
ansible-playbook site.yml --tags "config"    # Run tagged tasks
ansible-playbook site.yml --skip-tags "test" # Skip tagged tasks
ansible-playbook site.yml --list-tasks       # List all tasks
ansible-playbook site.yml --list-hosts       # List target hosts
ansible-playbook site.yml --syntax-check     # Validate YAML
ansible-playbook site.yml --start-at-task "Copy config"  # Start at task
ansible-playbook site.yml --step             # Confirm each task
ansible-playbook site.yml -e "var=value"     # Extra variables

# ─── Vault Commands ───
ansible-vault create secrets.yml             # Create encrypted file
ansible-vault edit secrets.yml               # Edit encrypted file
ansible-vault view secrets.yml               # View encrypted file
ansible-vault encrypt file.yml               # Encrypt existing file
ansible-vault decrypt file.yml               # Decrypt file
ansible-vault rekey file.yml                 # Change password
ansible-vault encrypt_string 'password' --name 'var_name'  # Encrypt string

# ─── Galaxy Commands ───
ansible-galaxy init roles/myrole             # Create role scaffolding
ansible-galaxy install user.role             # Install from Galaxy
ansible-galaxy install -r requirements.yml   # Install from file
ansible-galaxy list                          # List installed roles
ansible-galaxy collection install ns.name    # Install collection

# ─── Inventory Commands ───
ansible-inventory --list                     # Full inventory as JSON
ansible-inventory --graph                    # Inventory tree view
ansible all --list-hosts                     # List all hosts

# ─── Doc Commands ─── 
ansible-doc apt                              # Module documentation
ansible-doc -l                               # List all modules
ansible-doc -l | grep docker                 # Search modules
```

---

## 📋 YAML Quick Reference

```yaml
# ─── Playbook Structure ───
---
- name: Play Name
  hosts: target_group
  become: true
  gather_facts: true
  vars:
    key: value
  vars_files:
    - vars/secrets.yml
  
  pre_tasks:
    - name: Runs before roles
  
  roles:
    - common
    - role: nginx
      vars:
        port: 8080
  
  tasks:
    - name: Task description
      module_name:
        param1: value1
        param2: value2
      when: condition
      loop: [item1, item2]
      notify: Handler Name
      register: result
      tags: [tag1, tag2]
      become: true
      ignore_errors: true
      no_log: true
      changed_when: false

  post_tasks:
    - name: Runs after all tasks/roles

  handlers:
    - name: Handler Name
      service:
        name: nginx
        state: restarted
```

---

## 🔧 Most Used Modules

| Module | Purpose | Example |
|--------|---------|---------|
| `apt` | Package mgmt (Debian) | `apt: name=nginx state=present` |
| `yum` | Package mgmt (RHEL) | `yum: name=httpd state=present` |
| `package` | OS-agnostic packages | `package: name=git state=present` |
| `service` | Manage services | `service: name=nginx state=started enabled=yes` |
| `copy` | Copy files | `copy: src=f.txt dest=/tmp/ mode=0644` |
| `template` | Deploy Jinja2 templates | `template: src=t.j2 dest=/etc/app.conf` |
| `file` | Manage files/dirs | `file: path=/opt/app state=directory mode=0755` |
| `lineinfile` | Edit single line | `lineinfile: path=/etc/hosts line='...'` |
| `user` | Manage users | `user: name=deploy groups=sudo shell=/bin/bash` |
| `group` | Manage groups | `group: name=developers state=present` |
| `git` | Clone repos | `git: repo=url dest=/opt/app version=main` |
| `command` | Run command (no shell) | `command: /opt/script.sh` |
| `shell` | Run command (with shell) | `shell: cat /etc/hosts \| grep web` |
| `debug` | Print info | `debug: msg="{{ variable }}"` |
| `stat` | File info | `stat: path=/etc/nginx/nginx.conf` |
| `uri` | HTTP requests | `uri: url=http://localhost status_code=200` |
| `cron` | Manage cron jobs | `cron: name="backup" minute=0 hour=2 job="/opt/backup.sh"` |
| `sysctl` | Kernel parameters | `sysctl: name=net.ipv4.ip_forward value='1'` |
| `ufw` | Firewall | `ufw: rule=allow port='80'` |
| `wait_for` | Wait for condition | `wait_for: port=8080 timeout=60` |
| `unarchive` | Extract archives | `unarchive: src=app.tar.gz dest=/opt/ remote_src=yes` |
| `pip` | Python packages | `pip: name=flask state=present` |
| `get_url` | Download files | `get_url: url=http://... dest=/tmp/file` |
| `authorized_key` | SSH keys | `authorized_key: user=deploy key="{{ ssh_key }}"` |

---

## 🧮 Variable Precedence (High → Low)

```
1. -e / --extra-vars (command line)     ← HIGHEST
2. set_fact / register
3. Task vars
4. Block vars
5. Role vars (vars/main.yml)
6. Play vars_files
7. Play vars
8. Host facts
9. host_vars/
10. group_vars/
11. Inventory vars
12. Role defaults (defaults/main.yml)   ← LOWEST
```

---

## 🎯 Jinja2 Quick Reference

```jinja
{# ─── Variables ─── #}
{{ variable }}
{{ dict.key }}
{{ list[0] }}

{# ─── Conditionals ─── #}
{% if condition %}...{% endif %}
{% if x %}...{% elif y %}...{% else %}...{% endif %}

{# ─── Loops ─── #}
{% for item in list %}
  {{ item }}
{% endfor %}

{# ─── Common Filters ─── #}
{{ var | default("value") }}     {# default value #}
{{ var | upper / lower }}        {# case conversion #}
{{ list | join(", ") }}          {# join list items #}
{{ list | length }}              {# count items #}
{{ var | int / float / bool }}   {# type conversion #}
{{ var | to_json / to_yaml }}    {# format conversion #}
{{ "text" | hash('sha256') }}    {# hash #}
{{ "text" | b64encode }}         {# base64 encode #}
{{ list | sort / unique }}       {# list operations #}
{{ path | basename / dirname }}  {# path operations #}
```

---

## 🗂️ Role Directory Structure

```
roles/rolename/
├── defaults/main.yml    ← Default variables (override me!)
├── vars/main.yml        ← Internal variables
├── tasks/main.yml       ← Main task list (entry point)
├── handlers/main.yml    ← Event-driven tasks
├── templates/*.j2       ← Jinja2 config templates
├── files/*              ← Static files to copy
└── meta/main.yml        ← Dependencies & metadata
```

---

## 📂 Project Layout (Production)

```
project/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── production/
│   │   ├── hosts
│   │   └── group_vars/
│   │       ├── all/
│   │       │   ├── vars.yml
│   │       │   └── vault.yml    ← Encrypted!
│   │       └── webservers.yml
│   └── staging/
│       └── ...
├── playbooks/
│   ├── site.yml
│   ├── deploy.yml
│   └── rollback.yml
└── roles/
    ├── common/
    ├── nginx/
    └── app/
```

---

## 🧠 Memory Shortcuts

### Architecture: **"CIA"**
```
C = Control Node (your machine)
I = Inventory (server list)
A = Agentless (SSH-based)
```

### Execution Flow: **"PIGTEC"**
```
P = Parse playbook (validate YAML)
I = Inventory (identify hosts)
G = Gather facts (setup module)
T = Tasks (run in order)
E = Execute handlers (at end of play)
C = Cleanup (delete temp files)
```

### Playbook Hierarchy: **"PPTH"**
```
P = Playbook (the file)
P = Play (targets hosts)
T = Task (one action)
H = Handler (conditional action)
```

### Variable Priority: **"CTPHGR"**
```
C = Command line (-e)     ← Highest
T = Task (set_fact)
P = Play (vars/vars_files)
H = Host (host_vars)
G = Group (group_vars)
R = Role defaults          ← Lowest
```

---

## 🚨 Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `UNREACHABLE!` | SSH connection failed | Check SSH keys, IP, port |
| `MODULE FAILURE` | Python not found on target | Install python3 on target |
| `Permission denied` | SSH key not authorized | `ssh-copy-id user@host` |
| `sudo: a password is required` | Needs sudo password | Use `-K` or configure NOPASSWD |
| `Syntax Error` | Invalid YAML | Check indentation, use `--syntax-check` |
| `variable undefined` | Variable not set | Check spelling, use `default()` filter |
| `Could not find role` | Role path wrong | Check `roles_path` in ansible.cfg |

---

## 🔗 Useful Resources

| Resource | URL |
|----------|-----|
| Official Docs | https://docs.ansible.com/ |
| Module Index | https://docs.ansible.com/ansible/latest/collections/ |
| Galaxy | https://galaxy.ansible.com/ |
| Best Practices | https://docs.ansible.com/ansible/latest/tips_tricks/ |
| Jinja2 Docs | https://jinja.palletsprojects.com/ |

---

**⬅️ Previous: [13 - Real-World Projects](./13-real-world-projects.md)** | **🏠 [Index](./00-ansible-index.md)**

**🎉 Congratulations! You've completed the Ansible learning guide!**
