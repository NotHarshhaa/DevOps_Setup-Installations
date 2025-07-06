# Ansible Playbook Templates and Best Practices Guide

![banner](https://developers.redhat.com/sites/default/files/2024_AnsibleLP_featured_image_GS_Ansible%20Playbooks_0.png)

## 1. Project Structure

### 1.1. Recommended Directory Layout
```plaintext
ansible-project/
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
├── roles/
│   ├── common/
│   ├── webserver/
│   └── database/
├── playbooks/
│   ├── site.yml
│   ├── webserver.yml
│   └── database.yml
├── group_vars/
│   └── all.yml
├── host_vars/
│   └── specific_host.yml
├── library/
├── filter_plugins/
└── ansible.cfg
```

### 1.2. Sample Inventory Structure

```yaml
# inventory/production/hosts.yml
all:
  children:
    webservers:
      hosts:
        web01:
          ansible_host: 192.168.1.101
        web02:
          ansible_host: 192.168.1.102
    dbservers:
      hosts:
        db01:
          ansible_host: 192.168.1.201
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

## 2. Common Playbook Templates

### 2.1. Web Server Setup

```yaml
# playbooks/webserver.yml
---
- name: Configure Web Servers
  hosts: webservers
  become: true
  
  vars:
    http_port: 80
    https_port: 443
    app_root: /var/www/html
  
  tasks:
    - name: Install NGINX
      apt:
        name: nginx
        state: present
        update_cache: yes
      
    - name: Create web root directory
      file:
        path: "{{ app_root }}"
        state: directory
        mode: '0755'
      
    - name: Configure NGINX
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart NGINX
      
    - name: Start NGINX Service
      service:
        name: nginx
        state: started
        enabled: yes
  
  handlers:
    - name: Restart NGINX
      service:
        name: nginx
        state: restarted
```

### 2.2. Database Server Setup

```yaml
# playbooks/database.yml
---
- name: Configure Database Servers
  hosts: dbservers
  become: true
  
  vars:
    mysql_root_password: "{{ vault_mysql_root_password }}"
    mysql_databases:
      - name: app_db
        encoding: utf8mb4
        collation: utf8mb4_unicode_ci
    
    mysql_users:
      - name: app_user
        password: "{{ vault_app_db_password }}"
        priv: "app_db.*:ALL"
        host: "%"
  
  tasks:
    - name: Install MySQL
      apt:
        name: 
          - mysql-server
          - python3-mysqldb
        state: present
        update_cache: yes
      
    - name: Start MySQL Service
      service:
        name: mysql
        state: started
        enabled: yes
      
    - name: Create MySQL databases
      mysql_db:
        name: "{{ item.name }}"
        encoding: "{{ item.encoding }}"
        collation: "{{ item.collation }}"
        state: present
      loop: "{{ mysql_databases }}"
      
    - name: Create MySQL users
      mysql_user:
        name: "{{ item.name }}"
        password: "{{ item.password }}"
        priv: "{{ item.priv }}"
        host: "{{ item.host }}"
        state: present
      loop: "{{ mysql_users }}"
```

## 3. Role Templates

### 3.1. Common Role Structure

```yaml
# roles/common/tasks/main.yml
---
- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600
  when: ansible_os_family == "Debian"

- name: Install common packages
  package:
    name:
      - vim
      - curl
      - git
      - htop
    state: present

- name: Configure timezone
  timezone:
    name: UTC

- name: Configure system limits
  template:
    src: limits.conf.j2
    dest: /etc/security/limits.conf
```

### 3.2. Application Deployment Role

```yaml
# roles/app_deploy/tasks/main.yml
---
- name: Clone application repository
  git:
    repo: "{{ app_repo }}"
    dest: "{{ app_path }}"
    version: "{{ app_version }}"
  notify: Restart application

- name: Install application dependencies
  pip:
    requirements: "{{ app_path }}/requirements.txt"
    virtualenv: "{{ venv_path }}"
    state: present

- name: Configure application
  template:
    src: app_config.j2
    dest: "{{ app_path }}/config.yml"
  notify: Restart application

- name: Setup systemd service
  template:
    src: app_service.j2
    dest: /etc/systemd/system/{{ app_name }}.service
  notify: 
    - Reload systemd
    - Restart application

handlers:
  - name: Reload systemd
    systemd:
      daemon_reload: yes

  - name: Restart application
    service:
      name: "{{ app_name }}"
      state: restarted
```

## 4. Security Best Practices

### 4.1. Ansible Vault Usage

```yaml
# Create encrypted variables
ansible-vault create group_vars/all/vault.yml

# Example vault content
mysql_root_password: supersecret
app_secret_key: verysecretkey

# Using encrypted variables
---
- hosts: dbservers
  vars_files:
    - group_vars/all/vault.yml
  tasks:
    - name: Setup MySQL root password
      mysql_user:
        name: root
        password: "{{ mysql_root_password }}"
```

### 4.2. SSH Hardening Role

```yaml
# roles/ssh_hardening/tasks/main.yml
---
- name: Configure SSH
  template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    validate: '/usr/sbin/sshd -t -f %s'
  notify: Restart SSH

- name: Ensure SSH service is running
  service:
    name: sshd
    state: started
    enabled: yes

handlers:
  - name: Restart SSH
    service:
      name: sshd
      state: restarted
```

## 5. Deployment Strategies

### 5.1. Rolling Updates

```yaml
# playbooks/rolling_update.yml
---
- name: Perform rolling update
  hosts: webservers
  serial: 2
  max_fail_percentage: 25
  
  tasks:
    - name: Remove server from load balancer
      haproxy:
        state: disabled
        host: "{{ inventory_hostname }}"
        backend: web-backend
        
    - name: Update application
      git:
        repo: "{{ app_repo }}"
        dest: "{{ app_path }}"
        version: "{{ app_version }}"
        
    - name: Wait for application to start
      wait_for:
        port: 8000
        timeout: 60
        
    - name: Add server back to load balancer
      haproxy:
        state: enabled
        host: "{{ inventory_hostname }}"
        backend: web-backend
```

### 5.2. Blue-Green Deployment

```yaml
# playbooks/blue_green_deploy.yml
---
- name: Deploy new version (Green)
  hosts: green
  tasks:
    - name: Deploy application
      include_role:
        name: app_deploy
        
    - name: Run smoke tests
      uri:
        url: "http://{{ inventory_hostname }}/health"
        return_content: yes
      register: health_check
      failed_when: "'OK' not in health_check.content"
        
- name: Switch traffic to Green
  hosts: loadbalancer
  tasks:
    - name: Update LB configuration
      template:
        src: lb_config.j2
        dest: /etc/haproxy/haproxy.cfg
      vars:
        active_backend: green
      notify: Reload HAProxy
```

## 6. Testing and Validation

### 6.1. Molecule Testing

```yaml
# molecule/default/molecule.yml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: instance
    image: ubuntu:20.04
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: testinfra
  
# molecule/default/tests/test_default.py
def test_nginx_is_installed(host):
    nginx = host.package("nginx")
    assert nginx.is_installed
    
def test_nginx_is_running(host):
    nginx = host.service("nginx")
    assert nginx.is_running
    assert nginx.is_enabled
```

### 6.2. Playbook Syntax Check

```yaml
# test_playbook.yml
---
- name: Verify playbook syntax
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Run syntax check
      command: ansible-playbook site.yml --syntax-check
      
    - name: Run lint check
      command: ansible-lint site.yml
```

## 7. Performance Optimization

### 7.1. Ansible Configuration

```ini
# ansible.cfg
[defaults]
inventory = ./inventory
remote_user = ansible
host_key_checking = False
forks = 20
pipelining = True
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 7200

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
control_path = %(directory)s/%%h-%%r
pipelining = True
```

### 7.2. Parallel Execution

```yaml
# playbooks/parallel_tasks.yml
---
- name: Execute tasks in parallel
  hosts: all
  strategy: free
  tasks:
    - name: Long running task 1
      command: /long_running_script1.sh
      async: 3600
      poll: 0
      register: task1
      
    - name: Long running task 2
      command: /long_running_script2.sh
      async: 3600
      poll: 0
      register: task2
      
    - name: Wait for tasks to complete
      async_status:
        jid: "{{ item.ansible_job_id }}"
      register: job_result
      until: job_result.finished
      retries: 30
      delay: 10
      with_items:
        - "{{ task1 }}"
        - "{{ task2 }}"
```

## 8. Monitoring and Logging

### 8.1. Logging Configuration

```yaml
# playbooks/configure_logging.yml
---
- name: Configure centralized logging
  hosts: all
  tasks:
    - name: Install rsyslog
      package:
        name: rsyslog
        state: present
        
    - name: Configure rsyslog
      template:
        src: rsyslog.conf.j2
        dest: /etc/rsyslog.conf
      notify: Restart rsyslog
      
    - name: Enable rsyslog service
      service:
        name: rsyslog
        state: started
        enabled: yes
```

### 8.2. Monitoring Setup

```yaml
# playbooks/monitoring.yml
---
- name: Setup monitoring
  hosts: all
  tasks:
    - name: Install node exporter
      unarchive:
        src: https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
        dest: /usr/local/bin
        remote_src: yes
        
    - name: Configure node exporter service
      template:
        src: node_exporter.service.j2
        dest: /etc/systemd/system/node_exporter.service
      notify:
        - Reload systemd
        - Start node exporter
```

## 9. Best Practices

### 9.1. Code Organization
- Use roles for reusable configurations
- Keep playbooks focused and simple
- Use tags for selective execution
- Implement proper error handling
- Use variables for configuration

### 9.2. Security
- Use Ansible Vault for secrets
- Implement proper access controls
- Regular security updates
- Use secure communication
- Implement proper backup strategies

### 9.3. Performance
- Use fact caching
- Implement parallel execution
- Optimize module usage
- Use appropriate strategies
- Regular performance monitoring

## 10. Troubleshooting

### 10.1. Common Issues
1. SSH connectivity problems
2. Privilege escalation issues
3. Module failures
4. Variable scope problems
5. Task timing out

### 10.2. Debugging Tips
```bash
# Enable verbose output
ansible-playbook playbook.yml -vvv

# Check syntax
ansible-playbook playbook.yml --syntax-check

# Run in check mode
ansible-playbook playbook.yml --check

# List hosts that would be affected
ansible-playbook playbook.yml --list-hosts

# Step-by-step execution
ansible-playbook playbook.yml --step
``` 