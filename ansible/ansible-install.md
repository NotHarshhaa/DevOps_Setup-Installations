# Ansible Installation and Configuration Guide

![Ansible](https://www.ansible.com/hubfs/2019_Assets/Red_Hat_Ansible_Automation_Platform-Logo-RGB_White.png)

Comprehensive guide for installing and configuring Ansible across multiple operating systems with best practices and advanced configurations.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Installation](#2-installation)
3. [Configuration](#3-configuration)
4. [Inventory Management](#4-inventory-management)
5. [SSH Configuration](#5-ssh-configuration)
6. [Testing](#6-testing)
7. [Advanced Configuration](#7-advanced-configuration)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Introduction

### 1.1. What is Ansible?

Ansible is an open-source automation tool that simplifies configuration management, application deployment, and task automation. It uses a simple, agentless architecture and SSH to communicate with managed nodes.

### 1.2. Key Features
- **Agentless**: No agents required on managed nodes
- **Idempotent**: Safe to run multiple times
- **Simple YAML**: Easy to read and write
- **Push-based**: Executes tasks from control node
- **Cross-platform**: Works with Linux, Windows, macOS, and network devices

### 1.3. System Requirements
- Control Node: Python 3.8+ (Python 3.10+ recommended)
- Managed Nodes: Python 2.7+ or Python 3.5+
- SSH access to managed nodes
- Sudo/root access for privileged operations

---

## 2. Installation

### 2.1. Linux Installation

#### Ubuntu/Debian

**Method 1: Using PPA (Recommended)**
```bash
# Update package list
sudo apt update

# Install prerequisites
sudo apt install -y software-properties-common

# Add Ansible PPA
sudo add-apt-repository --yes --update ppa:ansible/ansible

# Install Ansible
sudo apt install -y ansible

# Verify installation
ansible --version
```

**Method 2: Using pip (Latest Version)**
```bash
# Install Python and pip
sudo apt install -y python3 python3-pip python3-venv

# Create virtual environment (optional but recommended)
python3 -m venv ~/.venv/ansible
source ~/.venv/ansible/bin/activate

# Install Ansible
pip install ansible-core

# Verify installation
ansible --version
```

**Method 3: Using apt (Stable Version)**
```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

#### CentOS/RHEL/Rocky Linux

**Method 1: Using EPEL Repository**
```bash
# Enable EPEL repository
sudo yum install -y epel-release

# Install Ansible
sudo yum install -y ansible

# Verify installation
ansible --version
```

**Method 2: Using pip**
```bash
# Install Python and pip
sudo yum install -y python3 python3-pip

# Install Ansible
pip3 install ansible-core --user

# Add to PATH
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Verify installation
ansible --version
```

#### Fedora

```bash
# Install Ansible
sudo dnf install -y ansible

# Verify installation
ansible --version
```

#### Arch Linux

```bash
# Install Ansible
sudo pacman -S ansible

# Verify installation
ansible --version
```

### 2.2. macOS Installation

**Method 1: Using Homebrew (Recommended)**
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Ansible
brew install ansible

# Verify installation
ansible --version
```

**Method 2: Using pip**
```bash
# Install Ansible
pip3 install ansible-core

# Verify installation
ansible --version
```

### 2.3. Windows Installation

**Method 1: Using Windows Subsystem for Linux (WSL)**
```powershell
# Enable WSL
wsl --install

# After reboot, open WSL and follow Linux installation instructions
```

**Method 2: Using Chocolatey**
```powershell
# Install Chocolatey if not already installed
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install Python
choco install python

# Install Ansible
pip install ansible-core

# Verify installation
ansible --version
```

### 2.4. Docker Installation

```bash
# Run Ansible in a Docker container
docker run --rm -it -v $(pwd):/workdir -w /workdir ansible/ansible:latest ansible --version

# Create an alias for convenience
alias ansible='docker run --rm -it -v $(pwd):/workdir -w /workdir ansible/ansible:latest ansible'
alias ansible-playbook='docker run --rm -it -v $(pwd):/workdir -w /workdir ansible/ansible:latest ansible-playbook'
```

### 2.5. Installation from Source

```bash
# Clone Ansible repository
git clone https://github.com/ansible/ansible.git
cd ansible

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install in development mode
pip install -r requirements.txt
source hacking/env-setup

# Verify installation
ansible --version
```

---

## 3. Configuration

### 3.1. Configuration File Locations

Ansible looks for configuration files in the following order (first found wins):
1. `ANSIBLE_CONFIG` environment variable
2. `./ansible.cfg` (current directory)
3. `~/.ansible.cfg` (home directory)
4. `/etc/ansible/ansible.cfg` (system-wide)

### 3.2. Basic Configuration

```ini
# ~/.ansible.cfg
[defaults]
# Inventory file location
inventory = ~/ansible/inventory

# Remote user
remote_user = ansible

# Disable SSH host key checking (not recommended for production)
host_key_checking = False

# Number of parallel processes
forks = 10

# SSH arguments
ssh_args = -o ControlMaster=auto -o ControlPersist=60s

# Privilege escalation
become = True
become_method = sudo
become_user = root

# Fact caching
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

# Display settings
display_skipped_hosts = False
display_ok_hosts = True

# Retry settings
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
control_path = %(directory)s/%%h-%%r
```

### 3.3. Environment Variables

```bash
# Set Ansible configuration file
export ANSIBLE_CONFIG=~/.ansible.cfg

# Set inventory file
export ANSIBLE_INVENTORY=~/ansible/inventory

# Set remote user
export ANSIBLE_REMOTE_USER=ansible

# Disable host key checking
export ANSIBLE_HOST_KEY_CHECKING=False

# Set number of forks
export ANSIBLE_FORKS=20

# Enable verbose output
export ANSIBLE_VERBOSITY=2
```

### 3.4. Python Interpreter Configuration

```ini
[defaults]
interpreter_python = auto_legacy_silent
```

Or specify per host in inventory:
```ini
[webservers]
web1 ansible_host=192.168.1.10 ansible_python_interpreter=/usr/bin/python3
web2 ansible_host=192.168.1.11 ansible_python_interpreter=/usr/bin/python3
```

---

## 4. Inventory Management

### 4.1. INI Format Inventory

```ini
# /etc/ansible/hosts
[webservers]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11
web3 ansible_host=192.168.1.12

[dbservers]
db1 ansible_host=192.168.1.20
db2 ansible_host=192.168.1.21

[all:vars]
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/ansible_key
ansible_python_interpreter=/usr/bin/python3

[webservers:vars]
http_port=80
https_port=443

[dbservers:vars]
mysql_port=3306
```

### 4.2. YAML Format Inventory (Recommended)

```yaml
# inventory.yml
all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.1.10
          ansible_user: ansible
        web2:
          ansible_host: 192.168.1.11
          ansible_user: ansible
      vars:
        http_port: 80
        https_port: 443
    
    dbservers:
      hosts:
        db1:
          ansible_host: 192.168.1.20
          ansible_user: ansible
        db2:
          ansible_host: 192.168.1.21
          ansible_user: ansible
      vars:
        mysql_port: 3306
  
  vars:
    ansible_ssh_private_key_file: ~/.ssh/ansible_key
    ansible_python_interpreter: /usr/bin/python3
```

### 4.3. Dynamic Inventory

#### AWS EC2 Dynamic Inventory

```bash
# Install boto3
pip install boto3 botocore

# Configure AWS credentials
aws configure

# Create AWS EC2 inventory script
cat > aws_ec2.yml <<EOF
plugin: aws_ec2
regions:
  - us-east-1
  - us-west-2
filters:
  tag:Environment:
    - production
  instance-state-name:
    - running
 keyed_groups:
  - key: tags.Environment
    prefix: env_
  - key: tags.Application
    prefix: app_
compose:
  ansible_host: private_ip_address
EOF

# Test dynamic inventory
ansible-inventory -i aws_ec2.yml --list
```

#### Dynamic Inventory with Script

```bash
# Install required packages
pip install ansible-builder

# Use inventory script
ansible -i /path/to/inventory_script.py all --list-hosts
```

### 4.4. Inventory Groups and Patterns

```bash
# List all hosts
ansible all --list-hosts

# List hosts in specific group
ansible webservers --list-hosts

# Use patterns
ansible 'webservers:&dbservers' --list-hosts  # Intersection
ansible 'webservers:!dbservers' --list-hosts  # Exclusion
ansible 'webservers:dbservers' --list-hosts   # Union
ansible '*.example.com' --list-hosts          # Wildcard
ansible '192.168.1.*' --list-hosts            # IP range
```

---

## 5. SSH Configuration

### 5.1. SSH Key Setup

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "ansible@control-node" -f ~/.ssh/ansible_key -N ""

# Copy public key to managed nodes
ssh-copy-id -i ~/.ssh/ansible_key.pub ansible@192.168.1.10
ssh-copy-id -i ~/.ssh/ansible_key.pub ansible@192.168.1.11

# Or use ansible ad-hoc command
ansible all -m authorized_key -a "user=ansible key='{{ lookup('file', '~/.ssh/ansible_key.pub') }}'" --ask-pass
```

### 5.2. SSH Config File

```ini
# ~/.ssh/config
Host web1
    HostName 192.168.1.10
    User ansible
    IdentityFile ~/.ssh/ansible_key
    StrictHostKeyChecking no

Host web2
    HostName 192.168.1.11
    User ansible
    IdentityFile ~/.ssh/ansible_key
    StrictHostKeyChecking no

Host db*
    HostName 192.168.1.20
    User ansible
    IdentityFile ~/.ssh/ansible_key
    StrictHostKeyChecking no
```

### 5.3. SSH Agent

```bash
# Start SSH agent
eval "$(ssh-agent -s)"

# Add key to agent
ssh-add ~/.ssh/ansible_key

# List keys in agent
ssh-add -l
```

### 5.4. SSH Connection Troubleshooting

```bash
# Test SSH connection
ssh -v ansible@192.168.1.10

# Test with Ansible
ansible webservers -m ping -vvv

# Check SSH config
ansible webservers -m debug -a "var=ansible_ssh_host_key_dsa_public_key"
```

---

## 6. Testing

### 6.1. Ad-hoc Commands

```bash
# Ping all hosts
ansible all -m ping

# Check uptime
ansible all -m command -a "uptime"

# Check disk space
ansible all -m shell -a "df -h"

# Check memory
ansible all -m shell -a "free -m"

# List packages
ansible webservers -m package -a "list=installed" | grep nginx

# Install package
ansible webservers -m apt -a "name=nginx state=present" --become

# Copy file
ansible webservers -m copy -a "src=/etc/hosts dest=/tmp/hosts backup=yes"

# Create directory
ansible webservers -m file -a "path=/tmp/test state=directory mode=0755"

# Check service status
ansible webservers -m service -a "name=nginx state=started"
```

### 6.2. Playbook Testing

```bash
# Syntax check
ansible-playbook playbook.yml --syntax-check

# List hosts
ansible-playbook playbook.yml --list-hosts

# List tasks
ansible-playbook playbook.yml --list-tasks

# Check mode (dry run)
ansible-playbook playbook.yml --check

# Diff mode
ansible-playbook playbook.yml --diff

# Start at specific task
ansible-playbook playbook.yml --start-at-task "Install Nginx"

# Run specific tags
ansible-playbook playbook.yml --tags "nginx,mysql"

# Skip specific tags
ansible-playbook playbook.yml --skip-tags "debug"
```

### 6.3. Connectivity Test Playbook

```yaml
# test-connectivity.yml
---
- name: Test Connectivity
  hosts: all
  gather_facts: no
  
  tasks:
    - name: Ping test
      ping:
      
    - name: Gather facts
      setup:
      
    - name: Display OS information
      debug:
        msg: "{{ inventory_hostname }} is running {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

```bash
# Run connectivity test
ansible-playbook test-connectivity.yml
```

---

## 7. Advanced Configuration

### 7.1. Ansible Collections

```bash
# Install collections
ansible-galaxy collection install community.general
ansible-galaxy collection install community.docker
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install kubernetes.core

# List installed collections
ansible-galaxy collection list

# Search for collections
ansible-galaxy collection search kubernetes

# Install specific version
ansible-galaxy collection install community.general:6.0.0

# Update collections
ansible-galaxy collection install --upgrade community.general

# Remove collection
ansible-galaxy collection remove community.general
```

### 7.2. Ansible Roles

```bash
# Install role from Galaxy
ansible-galaxy role install geerlingguy.nginx
ansible-galaxy role install geerlingguy.mysql

# Create role structure
ansible-galaxy role init myrole

# List installed roles
ansible-galaxy role list

# Search for roles
ansible-galaxy role search nginx

# Install requirements file
ansible-galaxy role install -r requirements.yml
```

**requirements.yml example:**
```yaml
roles:
  - name: geerlingguy.nginx
    version: 3.2.0
  - name: geerlingguy.mysql
    version: 4.3.0

collections:
  - name: community.general
    version: 6.0.0
  - name: community.docker
    version: 3.4.0
```

### 7.3. Ansible Vault

```bash
# Create encrypted file
ansible-vault create secret.yml

# Edit encrypted file
ansible-vault edit secret.yml

# Encrypt existing file
ansible-vault encrypt secret.yml

# Decrypt file
ansible-vault decrypt secret.yml

# Change password
ansible-vault rekey secret.yml

# View encrypted file
ansible-vault view secret.yml

# Encrypt string
ansible-vault encrypt_string 'mypassword' --name 'my_password'

# Run playbook with vault
ansible-playbook playbook.yml --ask-vault-pass

# Use vault password file
ansible-playbook playbook.yml --vault-password-file ~/.vault_pass

# Use multiple vault passwords
ansible-playbook playbook.yml --vault-id @prompt --vault-id prod@~/.prod_vault_pass
```

### 7.4. Ansible Galaxy

```bash
# Initialize new repository
ansible-galaxy init myproject

# Create collection
ansible-galaxy collection init mynamespace.mycollection

# Build collection
ansible-galaxy collection build

# Publish collection
ansible-galaxy collection publish mynamespace-mycollection-1.0.0.tar.gz --token YOUR_TOKEN
```

### 7.5. Performance Tuning

```ini
# ansible.cfg
[defaults]
# Increase forks for parallel execution
forks = 50

# Enable pipelining
pipelining = True

# Disable fact gathering if not needed
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

# Disable callback plugins for faster execution
bin_ansible_callbacks = False

[ssh_connection]
# Enable SSH pipelining
pipelining = True
control_path = %(directory)s/%%h-%%r
control_path_dir = /tmp/.ansible/cp

# Use SSH multiplexing
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o PreferredAuthentications=publickey
```

---

## 8. Troubleshooting

### 8.1. Common Issues

**Issue: SSH Connection Failed**
```bash
# Test SSH connection manually
ssh ansible@192.168.1.10

# Check SSH config
ansible webservers -m debug -a "var=ansible_ssh_host"

# Enable verbose output
ansible webservers -m ping -vvv
```

**Issue: Permission Denied**
```bash
# Check sudo access
ansible webservers -m shell -a "whoami" --become

# Use become method
ansible webservers -m ping --become --ask-become-pass
```

**Issue: Python Not Found**
```bash
# Specify Python interpreter
ansible webservers -m ping -e "ansible_python_interpreter=/usr/bin/python3"

# Install Python on remote host
ansible webservers -m raw -a "apt-get install -y python3" --become
```

**Issue: Module Not Found**
```bash
# Install required collections
ansible-galaxy collection install community.general

# Install required Python packages on managed nodes
ansible webservers -m pip -a "name=boto3 state=present" --become
```

### 8.2. Debugging Commands

```bash
# Enable verbose output
ansible-playbook playbook.yml -vvv

# Check syntax
ansible-playbook playbook.yml --syntax-check

# Run in check mode
ansible-playbook playbook.yml --check

# List affected hosts
ansible-playbook playbook.yml --list-hosts

# Step through tasks
ansible-playbook playbook.yml --step

# Debug specific task
ansible-playbook playbook.yml -e "ansible_debug=true"
```

### 8.3. Log Files

```bash
# Enable logging
[defaults]
log_path = /var/log/ansible.log

# Check logs
tail -f /var/log/ansible.log
```

---

## 9. Quick Reference

### 9.1. Common Commands

```bash
# Version check
ansible --version

# List hosts
ansible all --list-hosts

# Ping test
ansible all -m ping

# Run playbook
ansible-playbook playbook.yml

# Run with inventory
ansible-playbook -i inventory.yml playbook.yml

# Run with tags
ansible-playbook playbook.yml --tags "nginx"

# Check mode
ansible-playbook playbook.yml --check

# Verbose output
ansible-playbook playbook.yml -vvv
```

### 9.2. Useful Modules

```bash
# Package management
- apt: name=nginx state=present
- yum: name=nginx state=present
- pip: name=django state=present

# File operations
- copy: src=file dest=/path/ mode=0644
- template: src=template.j2 dest=/path/
- file: path=/path state=directory mode=0755

# Service management
- service: name=nginx state=started enabled=yes
- systemd: name=nginx state=started daemon_reload=yes

# User management
- user: name=ansible shell=/bin/bash groups=sudo
- authorized_key: user=ansible key="{{ lookup('file', 'key.pub') }}"

# Command execution
- command: /usr/bin/make_database
- shell: /usr/bin/make_database
- raw: yum install -y python
```

---

## 10. Best Practices

1. **Use version control** for all Ansible code
2. **Use roles** for reusable components
3. **Use variables** for configuration flexibility
4. **Use Ansible Vault** for sensitive data
5. **Test playbooks** in staging before production
6. **Use idempotent** operations
7. **Document your playbooks** with comments
8. **Use tags** for selective execution
9. **Implement error handling** with block/rescue
10. **Regular updates** of Ansible and collections

---

## References

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Ansible Collections](https://docs.ansible.com/ansible/latest/collections/index.html)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
