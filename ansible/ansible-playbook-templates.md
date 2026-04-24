# Ansible Playbook Templates and Best Practices Guide

![banner](https://www.ansible.com/hubfs/2019_Assets/Red_Hat_Ansible_Automation_Platform-Logo-RGB_White.png)

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

## 11. Modern Ansible Features (Ansible 2.15+)

### 11.1. FQCN (Fully Qualified Collection Name)

```yaml
# Old way (deprecated)
- name: Install package
  apt:
    name: nginx
    state: present

# New way (recommended)
- name: Install package
  ansible.builtin.apt:
    name: nginx
    state: present

# Using collections
- name: Create EC2 instance
  amazon.aws.ec2_instance:
    instance_type: t3.micro
    image_id: ami-12345678
```

### 11.2. Error Handling with Blocks

```yaml
- name: Deploy application with error handling
  block:
    - name: Stop application
      systemd:
        name: myapp
        state: stopped
    
    - name: Deploy new version
      git:
        repo: "{{ app_repo }}"
        dest: "{{ app_path }}"
        version: "{{ app_version }}"
    
    - name: Start application
      systemd:
        name: myapp
        state: started
  
  rescue:
    - name: Rollback deployment
      git:
        repo: "{{ app_repo }}"
        dest: "{{ app_path }}"
        version: "{{ previous_version }}"
    
    - name: Restart application
      systemd:
        name: myapp
        state: restarted
    
    - name: Send alert
      uri:
        url: "{{ alert_webhook }}"
        method: POST
        body: '{"status": "failed", "host": "{{ inventory_hostname }}"}'
  
  always:
    - name: Cleanup temporary files
      file:
        path: /tmp/deploy_temp
        state: absent
```

### 11.3. Loops and Iteration Improvements

```yaml
# Using loop instead of with_items
- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: alice, groups: sudo }
    - { name: bob, groups: developers }

# Using loop with register
- name: Check multiple services
  service_facts:
  register: services

- name: Display running services
  debug:
    msg: "{{ item }}"
  loop: "{{ services.ansible_facts.services | dict2items | selectattr('value.state', 'equalto', 'running') | list }}"

# Using until loop with retries
- name: Wait for service to be ready
  uri:
    url: "http://localhost:8080/health"
    return_content: yes
  register: result
  until: result.content == "OK"
  retries: 10
  delay: 5
```

### 11.4. Filters and Jinja2 Templates

```yaml
# Using filters
- name: Set variable with default
  debug:
    msg: "{{ my_var | default('default_value') }}"

# Using filters for lists
- name: Filter list
  debug:
    msg: "{{ my_list | selectattr('status', 'equalto', 'active') | list }}"

# Using filters for strings
- name: Convert to uppercase
  debug:
    msg: "{{ my_string | upper }}"

# Using custom filters in filter_plugins/
# filter_plugins/my_filters.py
def reverse_string(s):
    return s[::-1]

class FilterModule(object):
    def filters(self):
        return {
            'reverse_string': reverse_string
        }

# Usage in playbook
- name: Reverse string
  debug:
    msg: "{{ my_string | reverse_string }}"
```

### 11.5. Async Tasks

```yaml
- name: Run long task asynchronously
  command: /usr/bin/long_running_script
  async: 3600
  poll: 0
  register: long_task

- name: Do other tasks while waiting
  debug:
    msg: "Doing other work..."

- name: Wait for long task to complete
  async_status:
    jid: "{{ long_task.ansible_job_id }}"
  register: task_result
  until: task_result.finished
  retries: 30
  delay: 10
```

## 12. Ansible Collections Usage

### 12.1. AWS Collection

```yaml
- name: Create EC2 instance
  amazon.aws.ec2_instance:
    name: my-instance
    instance_type: t3.micro
    image_id: ami-12345678
    region: us-west-2
    vpc_subnet_id: subnet-12345
    security_groups:
      - sg-12345
    tags:
      Environment: production
      Application: myapp

- name: Create S3 bucket
  amazon.aws.s3_bucket:
    name: my-bucket
    region: us-west-2
    state: present
    public_access:
      block_public_acls: true
      block_public_policy: true
```

### 12.2. Kubernetes Collection

```yaml
- name: Create Kubernetes deployment
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
        namespace: default
      spec:
        replicas: 3
        selector:
          matchLabels:
            app: myapp
        template:
          metadata:
            labels:
              app: myapp
          spec:
            containers:
            - name: myapp
              image: myapp:latest
              ports:
              - containerPort: 8080

- name: Apply Helm chart
  kubernetes.core.helm:
    name: mychart
    chart_ref: stable/nginx
    release_namespace: default
    values:
      service:
        type: LoadBalancer
```

### 12.3. Docker Collection

```yaml
- name: Pull Docker image
  community.docker.docker_image:
    name: nginx:latest
    source: pull

- name: Run Docker container
  community.docker.docker_container:
    name: mycontainer
    image: nginx:latest
    state: started
    ports:
      - "80:80"
    volumes:
      - /data:/data

- name: Build Docker image
  community.docker.docker_image:
    name: myapp:latest
    build:
      path: /path/to/dockerfile
    source: build
```

### 12.4. Community General Collection

```yaml
- name: Manage PostgreSQL database
  community.postgresql.postgresql_db:
    name: mydb
    state: present

- name: Manage PostgreSQL user
  community.postgresql.postgresql_user:
    db: mydb
    name: myuser
    password: "{{ vault_db_password }}"
    priv: "ALL"

- name: Manage GitLab project
  community.general.gitlab_project:
    api_url: https://gitlab.example.com
    api_token: "{{ gitlab_token }}"
    name: myproject
    state: present
```

## 13. CI/CD Integration

### 13.1. GitHub Actions

```yaml
# .github/workflows/ansible-deploy.yml
name: Ansible Deploy

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'
    
    - name: Install Ansible
      run: pip install ansible-core ansible-lint
    
    - name: Install Galaxy collections
      run: ansible-galaxy collection install -r requirements.yml
    
    - name: Install Galaxy roles
      run: ansible-galaxy role install -r requirements.yml
    
    - name: Lint playbooks
      run: ansible-lint playbooks/
    
    - name: Run syntax check
      run: ansible-playbook playbooks/site.yml --syntax-check
    
    - name: Configure SSH
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy_key
        chmod 600 ~/.ssh/deploy_key
        ssh-keyscan -H ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts
    
    - name: Deploy to production
      env:
        ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD }}
      run: |
        echo $ANSIBLE_VAULT_PASSWORD > .vault_pass
        ansible-playbook playbooks/site.yml -i inventory/production --vault-password-file .vault_pass
```

### 13.2. Jenkins Pipeline

```groovy
pipeline {
    agent any
    
    environment {
        ANSIBLE_VAULT_PASSWORD = credentials('ansible-vault-password')
        SSH_PRIVATE_KEY = credentials('ssh-private-key')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'pip install ansible-core ansible-lint'
                sh 'ansible-galaxy collection install -r requirements.yml'
                sh 'ansible-galaxy role install -r requirements.yml'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'ansible-lint playbooks/'
            }
        }
        
        stage('Syntax Check') {
            steps {
                sh 'ansible-playbook playbooks/site.yml --syntax-check'
            }
        }
        
        stage('Deploy') {
            steps {
                withCredentials([
                    string(credentialsId: 'ansible-vault-password', variable: 'VAULT_PASS'),
                    sshUserPrivateKey(credentialsId: 'ssh-private-key', keyFileVariable: 'SSH_KEY_FILE')
                ]) {
                    sh '''
                        echo $VAULT_PASS > .vault_pass
                        chmod 600 $SSH_KEY_FILE
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible-playbook playbooks/site.yml -i inventory/production --vault-password-file .vault_pass --private-key $SSH_KEY_FILE
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh 'rm -f .vault_pass'
        }
    }
}
```

### 13.3. GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - deploy

variables:
  ANSIBLE_IMAGE: ansible/ansible:latest

before_script:
  - pip install ansible-core ansible-lint
  - ansible-galaxy collection install -r requirements.yml
  - ansible-galaxy role install -r requirements.yml

lint:
  stage: lint
  script:
    - ansible-lint playbooks/

syntax-check:
  stage: test
  script:
    - ansible-playbook playbooks/site.yml --syntax-check

deploy-staging:
  stage: deploy
  environment:
    name: staging
  script:
    - echo $VAULT_PASSWORD > .vault_pass
    - ansible-playbook playbooks/site.yml -i inventory/staging --vault-password-file .vault_pass
  only:
    - develop

deploy-production:
  stage: deploy
  environment:
    name: production
  script:
    - echo $VAULT_PASSWORD > .vault_pass
    - ansible-playbook playbooks/site.yml -i inventory/production --vault-password-file .vault_pass
  only:
    - main
  when: manual
```

## 14. Testing Strategies

### 14.1. Molecule Testing

```bash
# Install molecule
pip install molecule molecule-docker molecule-testinfra

# Initialize molecule for role
molecule init role myrole -d docker

# molecule/default/molecule.yml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: instance
    image: ubuntu:22.04
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
```

```python
# molecule/default/tests/test_default.py
import os

import testinfra.utils.ansible_runner

testinfra_hosts = testinfra.utils.ansible_runner.AnsibleRunner(
    os.environ['MOLECULE_INVENTORY_FILE']).get_hosts()


def test_nginx_is_installed(host):
    nginx = host.package("nginx")
    assert nginx.is_installed


def test_nginx_is_running(host):
    nginx = host.service("nginx")
    assert nginx.is_running
    assert nginx.is_enabled


def test_nginx_listening(host):
    assert host.socket("tcp://0.0.0.0:80").is_listening
```

### 14.2. Ansible Lint

```bash
# Install ansible-lint
pip install ansible-lint

# Run linter
ansible-lint playbooks/

# .ansible-lint configuration
# .ansible-lint
profile: production
exclude_paths:
  - .cache/
  - .git/
warn_list:
  - experimental
  - ignore-errors
skip_list:
  - yaml[line-length]
  - name[missing]
```

### 14.3. Integration Tests

```yaml
# tests/integration/test_playbook.yml
---
- name: Integration Test
  hosts: localhost
  connection: local
  gather_facts: no
  
  tasks:
    - name: Test playbook syntax
      command: ansible-playbook playbooks/site.yml --syntax-check
      register: syntax_check
      failed_when: syntax_check.rc != 0
    
    - name: Test playbook execution
      command: ansible-playbook playbooks/site.yml -i inventory/test --check
      register: execution_check
      failed_when: execution_check.rc != 0
```

## 15. Performance Optimization

### 15.1. Strategy Plugins

```yaml
# Use free strategy for parallel execution
- name: Deploy to all hosts in parallel
  hosts: all
  strategy: free
  tasks:
    - name: Deploy application
      include_role:
        name: app_deploy

# Use linear strategy (default)
- name: Deploy sequentially
  hosts: all
  strategy: linear
  tasks:
    - name: Deploy application
      include_role:
        name: app_deploy
```

### 15.2. Fact Caching

```ini
# ansible.cfg
[defaults]
fact_caching = redis
fact_caching_timeout = 3600
fact_caching_connection = localhost:6379:0
```

```yaml
# Disable fact gathering when not needed
- name: Run without facts
  hosts: all
  gather_facts: no
  tasks:
    - name: Simple task
      debug:
        msg: "Hello World"
```

### 15.3. SSH Optimization

```ini
# ansible.cfg
[ssh_connection]
pipelining = True
control_path = %(directory)s/%%h-%%r
control_path_dir = /tmp/.ansible/cp
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o PreferredAuthentications=publickey
```

## 16. Security Best Practices

### 16.1. Ansible Vault

```yaml
# Use vault for secrets
- name: Configure database
  template:
    src: database.conf.j2
    dest: /etc/database.conf
    mode: '0600'
  vars:
    db_password: "{{ vault_db_password }}"
```

### 16.2. No_log for Sensitive Data

```yaml
- name: Create user with password
  user:
    name: "{{ username }}"
    password: "{{ password }}"
  no_log: true
```

### 16.3. Become Method

```yaml
# Use become for privilege escalation
- name: Install package
  apt:
    name: nginx
    state: present
  become: true
  become_method: sudo
  become_user: root
```

## 17. Monitoring and Logging

### 17.1. Callback Plugins

```ini
# ansible.cfg
[defaults]
callback_whitelist = timer, profile_tasks, yaml
```

```yaml
# Custom callback plugin
# callback_plugins/my_callback.py
from ansible.plugins.callback import CallbackBase

class CallbackModule(CallbackBase):
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'notification'
    CALLBACK_NAME = 'my_callback'
    
    def v2_playbook_on_task_start(self, task, is_conditional):
        self._display.display(f"Starting task: {task.name}", color='blue')
```

### 17.2. ARA (Ansible Run Analysis)

```bash
# Install ARA
pip install ara

# Configure ansible.cfg
[defaults]
callback_plugins = /usr/local/lib/python3.10/site-packages/ara/plugins/callback
action_plugins = /usr/local/lib/python3.10/site-packages/ara/plugins/action

# Start ARA server
ara-manage runserver
```

## 18. Best Practices Summary

1. **Use FQCN** for all modules
2. **Use collections** for extended functionality
3. **Implement proper error handling** with blocks
4. **Use Ansible Vault** for sensitive data
5. **Test playbooks** with molecule and ansible-lint
6. **Use idempotent** operations
7. **Implement CI/CD** integration
8. **Use fact caching** for performance
9. **Document your code** with comments
10. **Use tags** for selective execution
11. **Implement proper logging** and monitoring
12. **Regular updates** of Ansible and collections

## References

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Ansible Collections](https://docs.ansible.com/ansible/latest/collections/index.html)
- [Molecule Documentation](https://molecule.readthedocs.io/)
- [Ansible Lint](https://ansible-lint.readthedocs.io/)