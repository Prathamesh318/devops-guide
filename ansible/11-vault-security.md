# 🔒 Chapter 11: Vault & Security

> **"Never commit passwords in plain text. Ansible Vault exists for exactly this."**

---

## 🤔 Why Does Vault Exist?

```
Problem:
  # group_vars/databases.yml
  db_password: SuperSecret123!        ← IN PLAIN TEXT!
  api_key: sk-12345-abcde            ← IN PLAIN TEXT!
  
  Now imagine this file is in Git:
    → Anyone with repo access sees your passwords
    → Leaked credentials = security disaster
    → Compliance violations (PCI, SOC2, GDPR)

Solution — Ansible Vault:
  → ENCRYPT sensitive files or variables
  → Encrypted files can safely live in Git
  → Only people with the vault password can decrypt
  → Everything else works exactly the same
```

---

## 🔑 Basic Vault Operations

### Create an Encrypted File

```bash
# Create a new encrypted file (opens editor)
ansible-vault create secrets.yml

# You'll be prompted:
# New Vault password: ********
# Confirm New Vault password: ********

# Your editor opens. Type your secrets:
---
db_password: SuperSecret123!
api_key: sk-12345-abcde
jwt_secret: my-jwt-secret-token

# Save and exit. The file is now encrypted!
```

### View the Encrypted File

```bash
# What the file looks like on disk:
cat secrets.yml

# Output (encrypted!):
$ANSIBLE_VAULT;1.1;AES256
39636230396162316561646330313038383738356636383666383265356534313635356435
61376262613836383232303064653031666535353536313830610a6230656233656266323
...

# Decrypt and view:
ansible-vault view secrets.yml
# Enter vault password to see contents
```

### Edit an Encrypted File

```bash
ansible-vault edit secrets.yml
# Decrypts → opens editor → re-encrypts on save
```

### Encrypt an Existing File

```bash
# You have a plain-text file you want to encrypt:
ansible-vault encrypt group_vars/production/secrets.yml
```

### Decrypt a File (Back to Plain Text)

```bash
# ⚠️ Be careful — makes the file readable!
ansible-vault decrypt secrets.yml
```

### Change Vault Password

```bash
ansible-vault rekey secrets.yml
# Enter: old password → new password
```

---

## ▶️ Using Vault with Playbooks

### Method 1: vars_files

```yaml
# playbook.yml
---
- name: Deploy application
  hosts: webservers
  become: true
  vars_files:
  - vars/config.yml                    # ← Plain text config
  - vars/secrets.yml                   # ← Vault encrypted!

  tasks:
  - name: Configure database
    template:
      src: db-config.j2
      dest: /etc/app/db.conf
    # Uses {{ db_password }} from secrets.yml — Ansible decrypts automatically!
```

```bash
# Run with vault password prompt
ansible-playbook playbook.yml --ask-vault-pass

# Run with password file
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# Run with environment variable
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook playbook.yml
```

### Method 2: Encrypted Variables Only (Inline)

```bash
# Encrypt just a single value (not the whole file!)
ansible-vault encrypt_string 'SuperSecret123!' --name 'db_password'

# Output:
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  39636230396162316561...
```

```yaml
# group_vars/production.yml — Mix of plain and encrypted!
---
# Plain text (not sensitive)
app_name: myapp
app_port: 8080
environment: production

# Encrypted (sensitive!)
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  39636230396162316561646330313038383738356636383666383265356534313635
  356435613762626138363832323030646530316665353533363138306130623065
  ...

api_key: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  61376262613836383232303064653031666535353536313830610a6230656233656
  ...
```

### Method 3: Vault Password File

```bash
# Create a password file
echo "MyVaultPassword123" > ~/.vault_pass
chmod 600 ~/.vault_pass                   # Only you can read!

# Add to ansible.cfg so you never have to type it
[defaults]
vault_password_file = ~/.vault_pass
```

⚠️ **NEVER commit the vault password file to Git!**
```bash
# Add to .gitignore
echo ".vault_pass" >> .gitignore
echo "*.vault_pass" >> .gitignore
```

---

## 🏗️ Best Practices for Vault

### Pattern 1: Separate Secrets File

```
group_vars/
├── all/
│   ├── vars.yml              ← Plain text: app_port, app_name
│   └── vault.yml             ← Encrypted: passwords, API keys
├── webservers/
│   ├── vars.yml              ← Plain text: nginx_port
│   └── vault.yml             ← Encrypted: ssl_key
└── databases/
    ├── vars.yml              ← Plain text: db_port
    └── vault.yml             ← Encrypted: db_password
```

### Pattern 2: Prefix Vault Variables

```yaml
# group_vars/databases/vault.yml (encrypted)
---
vault_db_password: SuperSecret123!
vault_db_root_password: RootPass456!

# group_vars/databases/vars.yml (plain text)
---
db_password: "{{ vault_db_password }}"
db_root_password: "{{ vault_db_root_password }}"

# Why? Now you can 'grep' for any variable and find where it's defined
# grep "db_password" → finds it in vars.yml
# vars.yml references vault.yml → you know it's encrypted
```

### Pattern 3: Multiple Vault IDs

```bash
# Different passwords for different environments!
ansible-playbook site.yml \
  --vault-id prod@~/.vault_pass_prod \
  --vault-id dev@~/.vault_pass_dev

# Encrypt with specific vault ID
ansible-vault encrypt --vault-id prod@prompt secrets-prod.yml
ansible-vault encrypt --vault-id dev@prompt secrets-dev.yml
```

---

## 🛡️ General Security Best Practices

### 1. SSH Security

```yaml
# Enforce SSH key-based auth
- name: Disable password authentication
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PasswordAuthentication'
    line: 'PasswordAuthentication no'
  notify: restart sshd

- name: Disable root login
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'
  notify: restart sshd
```

### 2. Minimal Privileges

```yaml
# Don't run everything as root!
- name: App tasks (no root needed)
  hosts: webservers
  become: false                        # ← Regular user

  tasks:
  - name: Deploy code
    git:
      repo: https://github.com/company/app
      dest: /home/deploy/app

# Only become root when necessary
  - name: Restart service (needs root)
    service:
      name: myapp
      state: restarted
    become: true                       # ← Root ONLY for this task
```

### 3. no_log — Hide Sensitive Output

```yaml
- name: Set database password
  mysql_user:
    name: admin
    password: "{{ db_password }}"
  no_log: true                         # ← Prevents password appearing in logs!

# Without no_log:
# TASK [Set database password] ****
# changed: [db1] => {"password": "SuperSecret123!"}  ← EXPOSED!

# With no_log:
# TASK [Set database password] ****
# changed: [db1] => {"censored": "output hidden due to no_log"}  ← SAFE!
```

### 4. File Permissions

```yaml
- name: Deploy secret config
  template:
    src: secret.conf.j2
    dest: /etc/app/secret.conf
    owner: appuser
    group: appgroup
    mode: '0600'                       # ← Only owner can read!
```

---

## 💡 Key Takeaways

```
1. Vault = encrypt sensitive data (passwords, keys, certificates)
2. Encrypted files can safely live in Git
3. ansible-vault create/edit/view/encrypt/decrypt
4. encrypt_string = encrypt single values inline
5. Vault password file = automate without typing password
6. NEVER commit vault password file to Git!
7. Pattern: separate vault.yml + vars.yml with vault_ prefix
8. no_log: true = hide sensitive task output
9. Always use minimum privileges (become only when needed)
10. File permissions: mode 0600 for secrets
```

---

**⬅️ Previous: [10 - Templates & Jinja2](./10-templates-jinja2.md)** | **Next: [12 - Advanced Patterns](./12-advanced-patterns.md)** ➡️
