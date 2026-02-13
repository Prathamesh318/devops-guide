# 🎯 Chapter 12: Advanced Patterns

> **"These are the techniques that separate a beginner from a production-ready Ansible engineer."**

---

## 🔀 Delegation — Run Task on a Different Host

### Why?

```
Problem: You're configuring web servers, but you need to
remove them from the load balancer FIRST (on the LB server)

Solution → delegate_to: run THIS task on THAT other host
```

```yaml
- name: Deploy web application
  hosts: webservers
  serial: 1                              # One server at a time (rolling)

  tasks:
  - name: Remove from load balancer     # ← Run on LB, not web server!
    command: /usr/local/bin/lb-remove {{ inventory_hostname }}
    delegate_to: lb1.example.com

  - name: Update application
    git:
      repo: https://github.com/company/app
      dest: /opt/app
    notify: Restart app

  - name: Wait for app to be healthy
    uri:
      url: "http://{{ inventory_hostname }}:8080/health"
      status_code: 200
    register: health
    until: health.status == 200
    retries: 30
    delay: 5

  - name: Add back to load balancer    # ← Run on LB again!
    command: /usr/local/bin/lb-add {{ inventory_hostname }}
    delegate_to: lb1.example.com

# Special: delegate to localhost (run on control node)
  - name: Send Slack notification
    uri:
      url: https://hooks.slack.com/services/xxx
      method: POST
      body_format: json
      body: '{"text": "Deployed to {{ inventory_hostname }}"}'
    delegate_to: localhost
```

---

## 🔄 Serial — Rolling Updates

```yaml
# Without serial: ALL servers updated at once
# → If broken = ENTIRE service is down!

# With serial: Update in batches
- name: Rolling deployment
  hosts: webservers  # 10 servers
  serial: 2          # 2 at a time

# Execution:
# Batch 1: web1, web2  → deploy → verify → ✅
# Batch 2: web3, web4  → deploy → verify → ✅
# Batch 3: web5, web6  → deploy → verify → ✅
# ... and so on

# Percentage-based:
  serial: "25%"        # 25% of hosts at a time

# Progressive (start slow, go faster):
  serial:
  - 1                  # First: just 1 server (canary)
  - 5                  # Then: 5 servers
  - "50%"             # Then: 50% of remaining
```

### Serial + max_fail_percentage

```yaml
- name: Safe rolling deployment
  hosts: webservers
  serial: 3
  max_fail_percentage: 30              # ← Abort if >30% of batch fails

  tasks:
  - name: Deploy
    shell: /opt/deploy.sh

  - name: Health check
    uri:
      url: "http://localhost:8080/health"
      status_code: 200

# If 2 out of 3 servers fail (66%) → exceeds 30% → ABORT deployment!
# Remaining servers are NOT touched → protected!
```

---

## 🏷️ Tags — Run Only What You Need

```yaml
- name: Full server setup
  hosts: all
  tasks:
  - name: Install packages
    apt: name={{ item }} state=present
    loop: [nginx, python3, git]
    tags: [packages, setup]

  - name: Deploy config
    template: src=app.conf.j2 dest=/etc/app.conf
    tags: [config]

  - name: Start service
    service: name=myapp state=started
    tags: [service, config]

  - name: Run smoke test
    uri: url=http://localhost:8080/health
    tags: [test, verify]
```

```bash
# Run only configuration tasks
ansible-playbook site.yml --tags "config"

# Run everything EXCEPT tests
ansible-playbook site.yml --skip-tags "test"

# Run packages and config
ansible-playbook site.yml --tags "packages,config"
```

---

## 🚨 Error Handling

### ignore_errors

```yaml
- name: Check if service exists (might not)
  command: systemctl status myapp
  register: result
  ignore_errors: true                  # ← Don't stop if this fails

- name: Install if not found
  apt: name=myapp state=present
  when: result.rc != 0
```

### failed_when — Custom Failure Conditions

```yaml
- name: Run migration
  command: /opt/app/migrate.sh
  register: migration
  failed_when:
  - "'ERROR' in migration.stdout"
  - migration.rc != 0

# Default: task fails when rc != 0
# Custom: also fail if output contains "ERROR"
```

### changed_when — Custom Change Detection

```yaml
- name: Check current version
  command: /opt/app/version.sh
  register: version_check
  changed_when: false                 # ← Never report as "changed"

- name: Run database backup
  command: /opt/backup.sh
  register: backup
  changed_when: "'New backup created' in backup.stdout"
```

### any_errors_fatal

```yaml
- name: Critical deployment
  hosts: webservers
  any_errors_fatal: true              # ← If ANY host fails, stop EVERYTHING

  tasks:
  - name: Database migration (must succeed on ALL)
    command: /opt/app/migrate.sh
```

---

## ⏱️ Async & Polling — Long-Running Tasks

```yaml
# Problem: Task takes 30 minutes. SSH timeout = 10 minutes.
# Solution: Run async!

- name: Long running database backup
  command: /opt/backup.sh --full
  async: 3600                          # ← Max runtime: 3600 seconds (1 hour)
  poll: 30                             # ← Check every 30 seconds

# Fire and forget (don't wait):
- name: Start background process
  command: /opt/long-job.sh
  async: 3600
  poll: 0                             # ← Don't wait at all!
  register: long_job

# Check later:
- name: Wait for background job
  async_status:
    jid: "{{ long_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 30
```

---

## 📋 Strategy — Execution Order

```yaml
# LINEAR (default): Task 1 on ALL hosts → Task 2 on ALL hosts → ...
- name: Play with linear strategy
  hosts: all
  strategy: linear

# FREE: Each host runs through all tasks independently
- name: Play with free strategy
  hosts: all
  strategy: free                     # ← Faster but harder to debug

# HOST_PINNED: Like free, but maintains host connection
- name: Play with host_pinned
  hosts: all
  strategy: host_pinned
```

---

## 📦 Dynamic Includes

```yaml
# Include different tasks based on OS
- name: Install packages
  include_tasks: "install_{{ ansible_os_family | lower }}.yml"
  # → Loads install_debian.yml OR install_redhat.yml dynamically!

# Include based on variable
- name: Deploy component
  include_tasks: "deploy_{{ app_component }}.yml"
  loop:
  - frontend
  - backend
  - database
  loop_control:
    loop_var: app_component
```

---

## 🏃 Run Once — Execute Task on Single Host Only

```yaml
- name: Database migration (run once!)
  hosts: webservers              # 10 servers
  tasks:
  - name: Run database migration
    command: /opt/app/migrate.sh
    run_once: true                    # ← Only runs on FIRST host!
    delegate_to: "{{ groups['databases'][0] }}"

  - name: Send deploy notification
    slack:
      msg: "Deployed version {{ app_version }}"
    run_once: true                    # ← Only send ONE notification
    delegate_to: localhost
```

---

## 🔧 Environment Variables

```yaml
- name: Run with custom environment
  hosts: all
  environment:                        # ← Set for all tasks in play
    http_proxy: http://proxy:8080
    https_proxy: http://proxy:8080
    no_proxy: localhost,127.0.0.1

  tasks:
  - name: Install packages (through proxy)
    apt:
      name: nginx
      state: present

  - name: Run app with custom env
    command: /opt/app/start.sh
    environment:                      # ← Override for this task
      APP_ENV: production
      DATABASE_URL: "postgresql://{{ db_host }}/{{ db_name }}"
      SECRET_KEY: "{{ vault_secret_key }}"
```

---

## ⏸️ Prompts — Interactive Playbooks

```yaml
- name: Deploy with confirmation
  hosts: webservers
  vars_prompt:
  - name: deploy_version
    prompt: "Which version to deploy?"
    default: "latest"
    private: false                    # Show input (not password)

  - name: confirm
    prompt: "Deploy {{ deploy_version }} to PRODUCTION? (yes/no)"
    private: false

  - name: db_password
    prompt: "Database password"
    private: true                     # ← Hide input (for passwords)

  tasks:
  - name: Abort if not confirmed
    fail:
      msg: "Deployment cancelled by user"
    when: confirm != "yes"

  - name: Deploy version
    debug:
      msg: "Deploying {{ deploy_version }}..."
```

---

## 📊 Wait_for — Wait for Conditions

```yaml
# Wait for port to be open
- name: Wait for app to start
  wait_for:
    port: 8080
    delay: 5                          # Wait 5 sec before first check
    timeout: 300                      # Max wait: 5 minutes

# Wait for file to exist
- name: Wait for lock file to disappear
  wait_for:
    path: /tmp/deploy.lock
    state: absent
    timeout: 120

# Wait for string in file
- name: Wait for app ready message
  wait_for:
    path: /var/log/app.log
    search_regex: "Application started"
    timeout: 60
```

---

## 💡 Key Takeaways

```
1. delegate_to = run task on a different host
2. serial = rolling updates (batch deployments)
3. max_fail_percentage = abort if too many failures
4. block/rescue/always = try/catch/finally error handling
5. ignore_errors = continue on failure
6. failed_when / changed_when = custom conditions
7. async + poll = long-running tasks without SSH timeout
8. run_once = execute on single host only
9. strategy: free = parallel host execution
10. wait_for = wait for ports, files, or conditions
```

---

**⬅️ Previous: [11 - Vault & Security](./11-vault-security.md)** | **Next: [13 - Real-World Projects](./13-real-world-projects.md)** ➡️
