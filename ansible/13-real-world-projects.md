# 🛠️ Chapter 13: Real-World Projects

> **"The best way to learn Ansible is to automate real things."**

---

## 📋 Project 1: LAMP Stack Automation

> **Deploy a full Linux + Apache + MySQL + PHP stack**

### Project Structure

```
lamp-project/
├── ansible.cfg
├── inventory
├── site.yml
├── roles/
│   ├── common/
│   │   └── tasks/main.yml
│   ├── apache/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   │   └── vhost.conf.j2
│   │   └── defaults/main.yml
│   ├── mysql/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── defaults/main.yml
│   └── php/
│       ├── tasks/main.yml
│       └── defaults/main.yml
└── group_vars/
    └── all.yml
```

### group_vars/all.yml

```yaml
---
# Domain configuration
domain: myapp.local
doc_root: /var/www/{{ domain }}

# MySQL configuration
mysql_root_password: "{{ vault_mysql_root_password }}"
mysql_db_name: myapp
mysql_db_user: appuser
mysql_db_password: "{{ vault_mysql_db_password }}"

# PHP configuration
php_version: "8.1"
```

### roles/common/tasks/main.yml

```yaml
---
- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install common packages
  apt:
    name:
    - vim
    - curl
    - wget
    - unzip
    - htop
    - ufw
    - git
    state: present

- name: Set timezone
  timezone:
    name: Asia/Kolkata

- name: Configure UFW defaults
  ufw:
    direction: incoming
    policy: deny

- name: Allow SSH
  ufw:
    rule: allow
    port: '22'
    proto: tcp

- name: Enable UFW
  ufw:
    state: enabled
```

### roles/apache/tasks/main.yml

```yaml
---
- name: Install Apache
  apt:
    name:
    - apache2
    - libapache2-mod-php{{ php_version }}
    state: present

- name: Create document root
  file:
    path: "{{ doc_root }}"
    state: directory
    owner: www-data
    group: www-data
    mode: '0755'

- name: Deploy virtual host
  template:
    src: vhost.conf.j2
    dest: /etc/apache2/sites-available/{{ domain }}.conf
  notify: Reload Apache

- name: Enable site
  command: a2ensite {{ domain }}.conf
  args:
    creates: /etc/apache2/sites-enabled/{{ domain }}.conf
  notify: Reload Apache

- name: Disable default site
  command: a2dissite 000-default.conf
  args:
    removes: /etc/apache2/sites-enabled/000-default.conf
  notify: Reload Apache

- name: Enable mod_rewrite
  apache2_module:
    name: rewrite
    state: present
  notify: Restart Apache

- name: Allow HTTP traffic
  ufw:
    rule: allow
    port: '80'

- name: Ensure Apache is running
  service:
    name: apache2
    state: started
    enabled: yes
```

### roles/apache/templates/vhost.conf.j2

```jinja
<VirtualHost *:80>
    ServerName {{ domain }}
    ServerAlias www.{{ domain }}
    DocumentRoot {{ doc_root }}

    <Directory {{ doc_root }}>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/{{ domain }}-error.log
    CustomLog ${APACHE_LOG_DIR}/{{ domain }}-access.log combined
</VirtualHost>
```

### roles/mysql/tasks/main.yml

```yaml
---
- name: Install MySQL
  apt:
    name:
    - mysql-server
    - mysql-client
    - python3-mysqldb
    state: present

- name: Start MySQL
  service:
    name: mysql
    state: started
    enabled: yes

- name: Set root password
  mysql_user:
    name: root
    password: "{{ mysql_root_password }}"
    login_unix_socket: /var/run/mysqld/mysqld.sock
  no_log: true

- name: Create application database
  mysql_db:
    name: "{{ mysql_db_name }}"
    state: present
    login_user: root
    login_password: "{{ mysql_root_password }}"

- name: Create application user
  mysql_user:
    name: "{{ mysql_db_user }}"
    password: "{{ mysql_db_password }}"
    priv: "{{ mysql_db_name }}.*:ALL"
    host: localhost
    state: present
    login_user: root
    login_password: "{{ mysql_root_password }}"
  no_log: true

- name: Remove anonymous users
  mysql_user:
    name: ''
    host_all: yes
    state: absent
    login_user: root
    login_password: "{{ mysql_root_password }}"
```

### roles/php/tasks/main.yml

```yaml
---
- name: Install PHP and extensions
  apt:
    name:
    - php{{ php_version }}
    - php{{ php_version }}-mysql
    - php{{ php_version }}-curl
    - php{{ php_version }}-gd
    - php{{ php_version }}-mbstring
    - php{{ php_version }}-xml
    - php{{ php_version }}-zip
    state: present
  notify: Restart Apache

- name: Deploy PHP info page (testing only!)
  copy:
    content: |
      <?php
      echo "<h1>LAMP Stack Working!</h1>";
      echo "<p>Server: <?php echo gethostname(); ?></p>";
      echo "<p>PHP Version: " . phpversion() . "</p>";
      phpinfo();
      ?>
    dest: "{{ doc_root }}/index.php"
    owner: www-data
    group: www-data
```

### site.yml — Master Playbook

```yaml
---
- name: Deploy LAMP Stack
  hosts: webservers
  become: true

  roles:
  - common
  - mysql
  - php
  - apache

  post_tasks:
  - name: Verify LAMP stack
    uri:
      url: "http://{{ inventory_hostname }}"
      status_code: 200
    delegate_to: localhost
```

```bash
# Deploy:
ansible-playbook site.yml

# Deploy to staging only:
ansible-playbook -i inventory/staging site.yml
```

---

## 📋 Project 2: Docker Host Setup

> **Install Docker + Docker Compose on multiple servers**

```yaml
# docker-setup.yml
---
- name: Setup Docker Host
  hosts: docker_hosts
  become: true

  vars:
    docker_users:
    - deploy
    - developer

  tasks:
  - name: Install prerequisites
    apt:
      name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
      state: present
      update_cache: yes

  - name: Add Docker GPG key
    apt_key:
      url: https://download.docker.com/linux/ubuntu/gpg
      state: present

  - name: Add Docker repository
    apt_repository:
      repo: "deb https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
      state: present

  - name: Install Docker
    apt:
      name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
      state: present
      update_cache: yes

  - name: Start Docker
    service:
      name: docker
      state: started
      enabled: yes

  - name: Add users to docker group
    user:
      name: "{{ item }}"
      groups: docker
      append: yes
    loop: "{{ docker_users }}"

  - name: Configure Docker daemon
    copy:
      content: |
        {
          "log-driver": "json-file",
          "log-opts": {
            "max-size": "10m",
            "max-file": "3"
          },
          "default-address-pools": [
            {"base": "172.20.0.0/16", "size": 24}
          ]
        }
      dest: /etc/docker/daemon.json
    notify: Restart Docker

  - name: Verify Docker installation
    command: docker --version
    changed_when: false
    register: docker_version

  - name: Show Docker version
    debug:
      msg: "Docker installed: {{ docker_version.stdout }}"

  handlers:
  - name: Restart Docker
    service:
      name: docker
      state: restarted
```

---

## 📋 Project 3: Kubernetes Cluster with Kubeadm

> **Bootstrap a K8s cluster using Ansible**

```yaml
# k8s-cluster.yml
---
# Play 1: Common setup on ALL nodes
- name: Prepare all K8s nodes
  hosts: k8s_all
  become: true

  tasks:
  - name: Disable swap
    command: swapoff -a
    changed_when: false

  - name: Remove swap from fstab
    lineinfile:
      path: /etc/fstab
      regexp: 'swap'
      state: absent

  - name: Load kernel modules
    modprobe:
      name: "{{ item }}"
    loop: [overlay, br_netfilter]

  - name: Configure sysctl
    sysctl:
      name: "{{ item.key }}"
      value: "{{ item.value }}"
      sysctl_file: /etc/sysctl.d/k8s.conf
    loop:
    - { key: net.bridge.bridge-nf-call-iptables, value: "1" }
    - { key: net.bridge.bridge-nf-call-ip6tables, value: "1" }
    - { key: net.ipv4.ip_forward, value: "1" }

  - name: Install containerd
    apt:
      name: containerd
      state: present
      update_cache: yes

  - name: Add Kubernetes GPG key
    apt_key:
      url: https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key

  - name: Add Kubernetes repository
    apt_repository:
      repo: "deb https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /"

  - name: Install K8s components
    apt:
      name:
      - kubelet
      - kubeadm
      - kubectl
      state: present

  - name: Hold K8s packages (prevent auto-update)
    dpkg_selections:
      name: "{{ item }}"
      selection: hold
    loop: [kubelet, kubeadm, kubectl]

# Play 2: Initialize master node
- name: Initialize Kubernetes Master
  hosts: k8s_master
  become: true

  tasks:
  - name: Initialize cluster
    command: kubeadm init --pod-network-cidr=10.244.0.0/16
    args:
      creates: /etc/kubernetes/admin.conf
    register: kubeadm_init

  - name: Create .kube directory
    file:
      path: /home/{{ ansible_user }}/.kube
      state: directory
      owner: "{{ ansible_user }}"

  - name: Copy admin config
    copy:
      src: /etc/kubernetes/admin.conf
      dest: /home/{{ ansible_user }}/.kube/config
      remote_src: yes
      owner: "{{ ansible_user }}"

  - name: Install Flannel CNI
    command: kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
    become_user: "{{ ansible_user }}"

  - name: Generate join command
    command: kubeadm token create --print-join-command
    register: join_command
    changed_when: false

  - name: Save join command
    set_fact:
      k8s_join_command: "{{ join_command.stdout }}"

# Play 3: Join worker nodes
- name: Join Worker Nodes
  hosts: k8s_workers
  become: true

  tasks:
  - name: Join cluster
    command: "{{ hostvars[groups['k8s_master'][0]]['k8s_join_command'] }}"
    args:
      creates: /etc/kubernetes/kubelet.conf
```

---

## 📋 Project 4: Complete Server Hardening

```yaml
# harden.yml
---
- name: Server Security Hardening
  hosts: all
  become: true

  vars:
    ssh_port: 22
    allowed_users: [deploy, admin]

  tasks:
  # SSH Hardening
  - name: SSH - Disable root login
    lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^PermitRootLogin'
      line: 'PermitRootLogin no'
    notify: Restart SSH

  - name: SSH - Disable password auth
    lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^PasswordAuthentication'
      line: 'PasswordAuthentication no'
    notify: Restart SSH

  - name: SSH - Limit max auth tries
    lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^MaxAuthTries'
      line: 'MaxAuthTries 3'
    notify: Restart SSH

  # Automatic security updates
  - name: Install unattended-upgrades
    apt:
      name: unattended-upgrades
      state: present

  - name: Enable automatic updates
    copy:
      content: |
        APT::Periodic::Update-Package-Lists "1";
        APT::Periodic::Unattended-Upgrade "1";
        APT::Periodic::AutocleanInterval "7";
      dest: /etc/apt/apt.conf.d/20auto-upgrades

  # Firewall
  - name: Configure UFW
    ufw:
      rule: allow
      port: "{{ item }}"
    loop:
    - '22'
    - '80'
    - '443'

  - name: Enable UFW with deny policy
    ufw:
      state: enabled
      policy: deny

  # Fail2ban
  - name: Install fail2ban
    apt:
      name: fail2ban
      state: present

  - name: Configure fail2ban
    copy:
      content: |
        [DEFAULT]
        bantime = 3600
        findtime = 600
        maxretry = 5

        [sshd]
        enabled = true
        port = {{ ssh_port }}
        filter = sshd
        logpath = /var/log/auth.log
      dest: /etc/fail2ban/jail.local
    notify: Restart Fail2ban

  handlers:
  - name: Restart SSH
    service: name=sshd state=restarted

  - name: Restart Fail2ban
    service: name=fail2ban state=restarted
```

---

## 🎯 Practice Order

```
1. LAMP Stack       → Learn roles, templates, handlers
2. Docker Setup     → Learn package management, repos, system config
3. K8s Cluster      → Learn multi-play, host groups, delegation
4. Server Hardening → Learn security, lineinfile, firewall
```

---

**⬅️ Previous: [12 - Advanced Patterns](./12-advanced-patterns.md)** | **Next: [14 - Cheat Sheet](./14-cheatsheet.md)** ➡️
