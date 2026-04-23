# Ansible Ubuntu Automation

Production-ready Ansible project for automating Ubuntu server provisioning and configuration.

## Folder Structure

```
ansible-ubuntu-automation/
├── ansible.cfg
├── README.md
├── group_vars/
│   └── all.yml
├── inventories/
│   └── production/
│       └── hosts.yml
├── playbooks/
│   ├── setup.yml
│   ├── security.yml
│   └── deploy.yml
└── roles/
    ├── common/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── handlers/
    │   │   └── main.yml
    │   └── tasks/
    │       └── main.yml
    ├── users/
    │   ├── defaults/
    │   │   └── main.yml
    │   └── tasks/
    │       └── main.yml
    ├── docker/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── handlers/
    │   │   └── main.yml
    │   └── tasks/
    │       └── main.yml
    └── nginx/
        ├── defaults/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        └── templates/
            └── reverse_proxy.j2
```

## Prerequisites

- Ansible installed on control node
- Target Ubuntu servers with SSH access
- SSH key pair for authentication

## Setup Instructions

1. Install Ansible:
   ```bash
   sudo apt update && sudo apt install ansible -y
   ```

2. Configure inventory:
   Edit `inventories/production/hosts.yml` with target server IPs.

3. Set SSH key:
   Replace `private_key_file` in `ansible.cfg` with your SSH private key path.

4. Set admin user SSH key:
   Update `admin_users[0].ssh_key` in `group_vars/all.yml` with your public key.

5. Run playbooks:
   ```bash
   # Initial setup
   ansible-playbook playbooks/setup.yml

   # Security hardening
   ansible-playbook playbooks/security.yml

   # Deploy Docker and Nginx
   ansible-playbook playbooks/deploy.yml
   ```

## Key Configuration Files

### ansible.cfg
```ini
[defaults]
inventory = ./inventories/production/hosts.yml
remote_user = ubuntu
private_key_file = ~/.ssh/id_rsa_ansible
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
bin_ansible_callbacks = True
roles_path = ./roles
collections_paths = ./collections

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
control_path_dir = ~/.ansible/cp
pipelining = True
```

### group_vars/all.yml
```yaml
# Shared variables for all hosts
ssh_port: 22
firewall_allowed_tcp_ports:
  - 22
  - 80
  - 443

# User configuration
default_user: ubuntu
admin_users:
  - name: "ansible-admin"
    ssh_key: "ssh-ed25519 AAAA... user@host"

# Docker configuration
docker_package_state: present
docker_compose_version: "v2.24.5"
docker_users:
  - "{{ default_user }}"

# Nginx configuration
nginx_listen_port: 80
nginx_proxy_target: "http://127.0.0.1:8080"
nginx_server_name: "_"

# System configuration
system_packages:
  - vim
  - curl
  - wget
  - git
  - ufw
  - python3
  - python3-pip
```

### inventories/production/hosts.yml
```yaml
---
all:
  children:
    webservers:
      hosts:
        ubuntu-prod-01:
          ansible_host: 192.168.1.100
          ansible_user: ubuntu
          ansible_python_interpreter: /usr/bin/python3
      vars:
        nginx_server_name: "prod.example.com"
        nginx_proxy_target: "http://127.0.0.1:3000"
```

### playbooks/setup.yml
```yaml
---
- name: Ubuntu Server Initial Setup
  hosts: all
  become: true
  roles:
    - common
```

### playbooks/security.yml
```yaml
---
- name: Security Hardening
  hosts: all
  become: true
  roles:
    - users
  tasks:
    - name: Configure UFW firewall rules
      ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: "{{ firewall_allowed_tcp_ports }}"
      notify: Enable UFW

    - name: Disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
        state: present
      notify: Restart SSH

    - name: Disable SSH password authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present
      notify: Restart SSH

    - name: Set SSH port
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^Port'
        line: 'Port {{ ssh_port }}'
        state: present
      notify: Restart SSH

  handlers:
    - name: Enable UFW
      ufw:
        state: enabled
        policy: deny

    - name: Restart SSH
      service:
        name: sshd
        state: restarted
```

### playbooks/deploy.yml
```yaml
---
- name: Deploy Docker and Nginx
  hosts: webservers
  become: true
  roles:
    - docker
    - nginx
```

### roles/common/tasks/main.yml
```yaml
---
- name: Update apt cache
  apt:
    update_cache: true
    cache_valid_time: "{{ update_cache_valid_time }}"

- name: Upgrade all packages
  apt:
    upgrade: "{{ upgrade_type }}"
    autoremove: true
    autoclean: true

- name: Install system packages
  apt:
    name: "{{ system_packages }}"
    state: present
  when: install_system_packages | bool
```

### roles/users/tasks/main.yml
```yaml
---
- name: Create admin user group
  group:
    name: admin
    state: present

- name: Create admin users
  user:
    name: "{{ item.name }}"
    shell: "{{ default_user_shell }}"
    groups: admin
    append: true
    state: present
  loop: "{{ admin_users }}"
  when: create_admin_users | bool

- name: Create .ssh directory for users
  file:
    path: "{{ ssh_key_directory | replace('{{ item.name }}', item.name) }}"
    state: directory
    owner: "{{ item.name }}"
    group: "{{ item.name }}"
    mode: '0700'
  loop: "{{ admin_users }}"
  when: create_admin_users | bool

- name: Add SSH authorized keys
  authorized_key:
    user: "{{ item.name }}"
    key: "{{ item.ssh_key }}"
    state: present
  loop: "{{ admin_users }}"
  when: create_admin_users | bool

- name: Allow admin group sudo without password
  lineinfile:
    path: /etc/sudoers
    regexp: '^%admin'
    line: '%admin ALL=(ALL) NOPASSWD: ALL'
    state: present
    validate: 'visudo -cf %s'
```

### roles/docker/tasks/main.yml
```yaml
---
- name: Install Docker prerequisites
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
    state: present
    update_cache: true

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker APT repository
  apt_repository:
    repo: "deb [arch={{ docker_apt_arch }}] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} {{ docker_apt_release_channel }}"
    state: present

- name: Install Docker Engine
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: "{{ docker_package_state }}"
    update_cache: true

- name: Start and enable Docker service
  service:
    name: docker
    state: started
    enabled: true

- name: Install Docker Compose
  get_url:
    url: "https://github.com/docker/compose/releases/download/{{ docker_compose_version }}/docker-compose-linux-x86_64"
    dest: /usr/local/bin/docker-compose
    mode: '0755'
    force: true

- name: Add users to docker group
  user:
    name: "{{ item }}"
    groups: docker
    append: true
  loop: "{{ docker_users }}"
```

### roles/nginx/tasks/main.yml
```yaml
---
- name: Install Nginx
  apt:
    name: nginx
    state: present
    update_cache: true

- name: Remove default Nginx site
  file:
    path: "{{ nginx_enabled_directory }}/{{ nginx_default_site }}"
    state: absent
  notify: Reload Nginx

- name: Deploy reverse proxy configuration
  template:
    src: reverse_proxy.j2
    dest: "{{ nginx_config_directory }}/reverse_proxy"
    mode: '0644'
  notify: Reload Nginx

- name: Enable reverse proxy site
  file:
    src: "{{ nginx_config_directory }}/reverse_proxy"
    dest: "{{ nginx_enabled_directory }}/reverse_proxy"
    state: link
  notify: Reload Nginx

- name: Start and enable Nginx
  service:
    name: nginx
    state: started
    enabled: true
```

### roles/nginx/templates/reverse_proxy.j2
```
server {
    listen {{ nginx_listen_port }};
    server_name {{ nginx_server_name }};

    location / {
        proxy_pass {{ nginx_proxy_target }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
