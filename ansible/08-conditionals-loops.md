# 🔀 Chapter 8: Conditionals & Loops

> **"Real automation needs logic — do this WHEN, do this FOR EACH."**

---

## 🤔 Why Do We Need Conditionals?

```
Without conditionals:
  → Install the same packages on Ubuntu AND CentOS? Impossible!
  → Ubuntu uses 'apt', CentOS uses 'yum'
  → You'd need separate playbooks for each OS

With conditionals:
  → "IF ubuntu, use apt. IF centos, use yum."
  → One playbook, all operating systems!
```

---

## 📌 The `when` Statement

### Basic Syntax

```yaml
- name: Install nginx on Ubuntu
  apt:
    name: nginx
    state: present
  when: ansible_distribution == "Ubuntu"       # ← Only runs on Ubuntu!

- name: Install nginx on CentOS
  yum:
    name: nginx
    state: present
  when: ansible_distribution == "CentOS"       # ← Only runs on CentOS!
```

### Comparison Operators

```yaml
# Equal
when: ansible_distribution == "Ubuntu"

# Not equal
when: ansible_os_family != "Windows"

# Greater than / Less than
when: ansible_memory_mb.real.total > 2048
when: ansible_processor_cores < 4

# Greater/Less than or equal
when: http_port >= 1024
when: max_retries <= 5

# In a list
when: ansible_distribution in ["Ubuntu", "Debian"]

# Not in a list
when: ansible_distribution not in ["Windows", "MacOSX"]

# Contains (string)
when: "'nginx' in installed_packages"

# Boolean
when: ssl_enabled                    # True = run
when: not debug_mode                 # Not True = run
when: ssl_enabled | bool             # Convert string to bool
```

### Multiple Conditions

```yaml
# AND (all conditions must be true)
- name: Install on Ubuntu 22.04 only
  apt:
    name: nginx
    state: present
  when:
  - ansible_distribution == "Ubuntu"          # Condition 1
  - ansible_distribution_version == "22.04"   # AND Condition 2

# OR (any condition can be true)
- name: Install on Debian-family
  apt:
    name: nginx
    state: present
  when: ansible_distribution == "Ubuntu" or ansible_distribution == "Debian"

# Complex logic
- name: Install on production Ubuntu with enough RAM
  apt:
    name: nginx
    state: present
  when:
  - ansible_distribution == "Ubuntu"
  - inventory_hostname in groups['production']
  - ansible_memory_mb.real.total >= 2048
```

### Conditionals with `register`

```yaml
- name: Check if config file exists
  stat:
    path: /etc/nginx/nginx.conf
  register: config_file

- name: Create config only if it doesn't exist
  copy:
    content: "# Default nginx config"
    dest: /etc/nginx/nginx.conf
  when: not config_file.stat.exists         # ← Use registered result!

# Common register checks:
#   result.rc == 0           → command succeeded
#   result.failed            → task failed
#   result.changed           → task made changes
#   result.stdout != ""      → output is not empty
#   result.stat.exists       → file exists (stat module)
```

### Conditional on Variable Existence

```yaml
# Check if variable is defined
- name: Use custom port if defined
  debug:
    msg: "Port: {{ custom_port }}"
  when: custom_port is defined

# Check if variable is NOT defined
- name: Use default port
  debug:
    msg: "Using default port 80"
  when: custom_port is not defined

# Check if variable is empty
- name: Skip if no users
  debug:
    msg: "Processing users"
  when: user_list | length > 0
```

---

## 🔄 Loops

### Why Do Loops Exist?

```
Without loops:
  - apt: name=nginx state=present
  - apt: name=python3 state=present
  - apt: name=git state=present
  - apt: name=curl state=present
  → Repetitive! 4 tasks for 4 packages

With loops:
  - apt: name="{{ item }}" state=present
    loop: [nginx, python3, git, curl]
  → ONE task, 4 packages!
```

### `loop` (Modern — Use This!)

```yaml
# Simple list loop
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
  - nginx
  - python3
  - git
  - curl

# Loop with variables
- name: Create users
  user:
    name: "{{ item }}"
    state: present
  loop: "{{ user_list }}"
```

### Loop over Dictionaries

```yaml
# Loop over list of dictionaries
- name: Create users with details
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    shell: "{{ item.shell }}"
    state: present
  loop:
  - { name: alice, groups: sudo, shell: /bin/bash }
  - { name: bob, groups: developers, shell: /bin/bash }
  - { name: charlie, groups: "sudo,developers", shell: /bin/zsh }
```

### Loop with `dict2items`

```yaml
vars:
  users:
    alice: sudo
    bob: developers
    charlie: docker

tasks:
- name: Create users
  user:
    name: "{{ item.key }}"
    groups: "{{ item.value }}"
  loop: "{{ users | dict2items }}"
  # item.key = alice, item.value = sudo
```

### Loop with Index

```yaml
- name: Show items with index
  debug:
    msg: "Item {{ index }}: {{ item }}"
  loop:
  - nginx
  - python3
  - git
  loop_control:
    index_var: index               # ← Get loop index
    # index starts at 0
```

### Loop Control

```yaml
- name: Install packages with custom label
  apt:
    name: "{{ item }}"
    state: present
  loop:
  - nginx
  - python3-pip
  - postgresql-client
  loop_control:
    label: "{{ item }}"            # ← What to show in output (default: entire item)
    pause: 2                        # ← Wait 2 seconds between iterations
    # extended: true               # ← Get extra vars (ansible_loop.first, last, etc.)
```

### Nested Loops with `subelements`

```yaml
vars:
  users:
  - name: alice
    ssh_keys:
    - ssh-rsa AAAA... alice@laptop
    - ssh-rsa BBBB... alice@desktop
  - name: bob
    ssh_keys:
    - ssh-rsa CCCC... bob@laptop

tasks:
- name: Add SSH keys for each user
  authorized_key:
    user: "{{ item.0.name }}"
    key: "{{ item.1 }}"
  loop: "{{ users | subelements('ssh_keys') }}"
  # item.0 = user dict, item.1 = each SSH key
```

### `until` — Retry Loop

```yaml
- name: Wait for service to be ready
  uri:
    url: http://localhost:8080/health
    status_code: 200
  register: result
  until: result.status == 200        # ← Keep trying until this is true
  retries: 30                        # ← Maximum attempts
  delay: 10                          # ← Seconds between attempts
  # Total max wait: 30 * 10 = 300 seconds = 5 minutes
```

---

## 🧱 Blocks — Group Tasks with Shared Logic

### Why Do Blocks Exist?

```
Without blocks:
  - task 1  when: ansible_os_family == "Debian"
  - task 2  when: ansible_os_family == "Debian"
  - task 3  when: ansible_os_family == "Debian"
  → Repeating the SAME condition on every task!

With blocks:
  block:
    - task 1
    - task 2
    - task 3
  when: ansible_os_family == "Debian"
  → ONE condition for ALL tasks!
```

### Block Syntax

```yaml
- name: Setup web server (Debian family)
  block:
  - name: Install nginx
    apt:
      name: nginx
      state: present

  - name: Start nginx
    service:
      name: nginx
      state: started

  - name: Open firewall
    ufw:
      rule: allow
      port: '80'
  when: ansible_os_family == "Debian"     # ← Applies to ALL tasks in block
  become: true                             # ← Also applies to all tasks
```

### Error Handling with block/rescue/always

```yaml
# Like try/catch/finally in programming!
- name: Deploy with error recovery
  block:
  # TRY — these tasks might fail
  - name: Pull latest code
    git:
      repo: https://github.com/company/app.git
      dest: /opt/myapp
      version: "{{ deploy_version }}"

  - name: Install dependencies
    pip:
      requirements: /opt/myapp/requirements.txt

  - name: Start application
    service:
      name: myapp
      state: restarted

  rescue:
  # CATCH — runs ONLY if block tasks fail
  - name: Rollback to previous version
    git:
      repo: https://github.com/company/app.git
      dest: /opt/myapp
      version: "{{ previous_version }}"

  - name: Restart with old version
    service:
      name: myapp
      state: restarted

  - name: Send notification
    mail:
      to: admin@example.com
      subject: "Deploy FAILED — rolled back to {{ previous_version }}"

  always:
  # FINALLY — runs ALWAYS regardless of success or failure
  - name: Clean up temp files
    file:
      path: /tmp/deploy-artifacts
      state: absent

  - name: Log deployment result
    shell: echo "Deploy at $(date) - {{ 'SUCCESS' if not ansible_failed_task is defined else 'FAILED' }}" >> /var/log/deploys.log
```

---

## 🔗 Combining Conditionals + Loops

```yaml
# Only install on Debian family
- name: Install packages (Debian only)
  apt:
    name: "{{ item }}"
    state: present
  loop:
  - nginx
  - python3
  - git
  when: ansible_os_family == "Debian"

# Conditional inside loop
- name: Create users only if active
  user:
    name: "{{ item.name }}"
    state: present
  loop:
  - { name: alice, active: true }
  - { name: bob, active: false }
  - { name: charlie, active: true }
  when: item.active                    # ← Check each item's 'active' flag
```

---

## 🏷️ Tags — Run Specific Parts of Playbook

```yaml
# tags.yml
- name: Full setup
  hosts: all
  become: true

  tasks:
  - name: Install packages
    apt:
      name: [nginx, python3]
      state: present
    tags:
    - packages
    - setup

  - name: Copy config
    copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    tags:
    - config

  - name: Start service
    service:
      name: nginx
      state: started
    tags:
    - service
    - config
```

```bash
# Run ONLY tasks with specific tags
ansible-playbook tags.yml --tags "config"
ansible-playbook tags.yml --tags "packages,service"

# Skip tasks with specific tags
ansible-playbook tags.yml --skip-tags "packages"

# List all available tags
ansible-playbook tags.yml --list-tags

# Special tags:
ansible-playbook tags.yml --tags "all"       # Run everything
ansible-playbook tags.yml --tags "tagged"     # Only tagged tasks
ansible-playbook tags.yml --tags "untagged"   # Only untagged tasks
```

---

## 💡 Key Takeaways

```
1. when = conditional execution (if/else)
2. loop = iterate over lists (for each)
3. register = capture task output for later use
4. block = group tasks with shared conditions
5. block/rescue/always = error handling (try/catch/finally)
6. until = retry loop with delay
7. Tags = run specific parts of a playbook
8. Loops + conditionals can be combined
9. Always prefer modules over shell commands with conditionals
```

---

**⬅️ Previous: [07 - Variables & Facts](./07-variables-facts.md)** | **Next: [09 - Roles & Galaxy](./09-roles-galaxy.md)** ➡️
