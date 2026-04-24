# AWX (Ansible Tower) Installation and Configuration Guide

![AWX](https://raw.githubusercontent.com/ansible/awx/devel/docs/logo/awx-logo.png)

Comprehensive guide for installing and configuring AWX (open-source Ansible Tower) for enterprise automation.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Installation Methods](#3-installation-methods)
4. [Docker Installation](#4-docker-installation)
5. [Kubernetes Installation](#5-kubernetes-installation)
6. [Operator Installation](#6-operator-installation)
7. [Post-Installation Configuration](#7-post-installation-configuration)
8. [User Management](#8-user-management)
9. [Project Setup](#9-project-setup)
10. [Best Practices](#10-best-practices)

---

## 1. Introduction

### 1.1. What is AWX?

AWX is the open-source version of Ansible Tower, providing a web-based UI, REST API, and task engine for Ansible. It enables teams to centrally manage and control Ansible automation.

### 1.2. Key Features
- **Web-based UI**: Visual dashboard for managing automation
- **REST API**: Programmatic access to all features
- **RBAC**: Role-based access control
- **Job Scheduling**: Schedule and automate playbook runs
- **Credential Management**: Secure storage of credentials
- **Inventory Management**: Dynamic and static inventory
- **Workflow Orchestration**: Complex multi-playbook workflows
- **Analytics and Reporting**: Job history and performance metrics

### 1.3. AWX vs Ansible Tower

| Feature | AWX | Ansible Tower |
|---------|-----|--------------|
| Cost | Free | Paid |
| Support | Community | Red Hat |
| Updates | Manual | Automatic |
| SLA | None | Included |
| Features | Full | Full + Enterprise |

---

## 2. Prerequisites

### 2.1. System Requirements

**Minimum Requirements:**
- CPU: 4 cores
- RAM: 8 GB
- Disk: 40 GB
- OS: RHEL 8+, Ubuntu 20.04+, or compatible

**Recommended Requirements:**
- CPU: 8 cores
- RAM: 16 GB
- Disk: 100 GB
- OS: RHEL 9 or Ubuntu 22.04

### 2.2. Software Requirements

```bash
# Docker Installation
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Python and pip
sudo apt install -y python3 python3-pip python3-venv

# Git
sudo apt install -y git

# Node.js (for AWX development)
sudo apt install -y nodejs npm
```

### 2.3. Network Requirements

- Ports: 80 (HTTP), 443 (HTTPS)
- Outbound internet access for container downloads
- DNS resolution for container registry

---

## 3. Installation Methods

AWX can be installed using several methods:

1. **Docker Compose**: Quick setup for development/testing
2. **Kubernetes Helm**: Production deployment on K8s
3. **AWX Operator**: Kubernetes-native installation
4. **Bare Metal**: Traditional installation

---

## 4. Docker Installation

### 4.1. Clone AWX Repository

```bash
# Clone AWX repository
git clone --depth 1 --branch 23.0.0 https://github.com/ansible/awx.git
cd awx/tools/docker
```

### 4.2. Configure AWX

```bash
# Copy inventory file
cp inventory inventory.backup

# Edit inventory file
nano inventory
```

**inventory file configuration:**
```ini
[all:vars]
awx_task_hostname=awx
awx_web_hostname=awxweb

postgres_data_dir=/var/lib/awx/pgdocker
host_port=80
host_port_ssl=443

docker_compose_dir=./
pg_username=awx
pg_password=awxpass
pg_database=awx
pg_port=5432

rabbitmq_user=awx
rabbitmq_password=awxpass
rabbitmq_erlang_cookie=cookiemaster

admin_user=admin
admin_password=password

create_preload_data=True
secret_key=awxsecret

project_data_dir=/var/lib/awx/projects
```

### 4.3. Generate Secret Key

```bash
# Generate secret key
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

Update the inventory file with the generated secret key.

### 4.4. Run AWX Installer

```bash
# Run the installer
ansible-playbook -i inventory install.yml

# This will take 10-15 minutes to complete
```

### 4.5. Verify Installation

```bash
# Check containers
docker-compose ps

# Check logs
docker-compose logs -f

# Access AWX UI
# Open browser: http://localhost
# Default credentials: admin / password
```

---

## 5. Kubernetes Installation

### 5.1. Using Helm Chart

```bash
# Add AWX Helm repository
helm repo add awx-operator https://ansible.github.io/awx-operator/
helm repo update

# Install AWX Operator
kubectl create namespace awx
helm install awx-operator awx-operator/awx-operator -n awx

# Wait for operator to be ready
kubectl wait --for=condition=available --timeout=60s deployment/awx-operator-controller-manager -n awx
```

### 5.2. Create AWX Instance

```yaml
# awx-instance.yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: loadbalancer
  ingress_type: none
  hostname: awx.example.com
  
  postgres_configuration:
    postgres_image: registry.redhat.io/rhel8/postgresql-13:1-54
    postgres_volume_storage_class: standard
    postgres_resource_requirements:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 1000m
        memory: 4Gi
  
  web_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi
  
  task_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi
  
  ee_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi
```

```bash
# Apply AWX instance
kubectl apply -f awx-instance.yaml -n awx

# Wait for AWX to be ready
kubectl wait --for=condition=Running --timeout=600s awx/awx -n awx

# Get AWX URL
kubectl get awx awx -n awx -o jsonpath='{.status.awxURL}'
```

### 5.3. Access AWX

```bash
# Get admin password
kubectl get secret awx-admin-password -n awx -o jsonpath='{.data.password}' | base64 -d

# Port forward to access locally
kubectl port-forward svc/awx -n awx 80:80

# Or use LoadBalancer
kubectl get svc awx -n awx
```

---

## 6. Operator Installation

### 6.1. Install AWX Operator

```bash
# Clone AWX operator repository
git clone --depth 1 --branch 2.10.0 https://github.com/ansible/awx-operator.git
cd awx-operator

# Install CRDs
kubectl apply -f config/crd/bases/

# Install RBAC
kubectl apply -f config/rbac/

# Install operator
kubectl apply -f config/manager/
```

### 6.2. Deploy AWX

```yaml
# awx-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: awx
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-configuration
  namespace: awx
type: Opaque
stringData:
  host: postgres
  port: "5432"
  database: awx
  username: awx
  password: awxpass
  type: managed
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  deployment_name: awx
  web_replicas: 1
  task_replicas: 1
  postgres_configuration_secret: postgres-configuration
```

```bash
# Apply deployment
kubectl apply -f awx-deployment.yaml

# Monitor deployment
kubectl get pods -n awx -w
```

---

## 7. Post-Installation Configuration

### 7.1. Initial Setup

1. **Access AWX UI**
   - Open browser: `http://your-awx-host`
   - Login with admin credentials

2. **Change Admin Password**
   - Navigate to User Management
   - Select admin user
   - Update password

3. **Configure License** (for Ansible Tower)
   - Navigate to Settings → License
   - Upload license file

### 7.2. Configure Settings

```bash
# Access AWX CLI
awx-cli login http://localhost --conf insecure

# Configure settings
awx-cli config settings set AWX_TASK_ENV["AWX_TASK_ENVIRONMENT"] "production"
awx-cli config settings set AWX_TASK_ENV["AWX_TASK_PROOTECT_ENABLED"] "true"
```

### 7.3. Configure Email

```yaml
# Navigate to Settings → Email
# Configure SMTP settings
smtp_host: smtp.gmail.com
smtp_port: 587
smtp_user: your-email@gmail.com
smtp_password: your-password
smtp_use_ssl: false
smtp_use_tls: true
```

### 7.4. Configure LDAP/AD

```yaml
# Navigate to Settings → LDAP
# Configure LDAP settings
ldap_uri: ldap://ldap.example.com
ldap_bind_dn: cn=admin,dc=example,dc=com
ldap_bind_password: password
ldap_search_dn: ou=users,dc=example,dc=com
ldap_user_search_filter: (uid=%(user)s)
ldap_group_search_filter: (objectclass=group)
```

---

## 8. User Management

### 8.1. Create Users

```bash
# Using AWX CLI
awx-cli user create --username john --email john@example.com --first-name John --last-name Doe

# Using API
curl -X POST http://localhost/api/v2/users/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john",
    "email": "john@example.com",
    "first_name": "John",
    "last_name": "Doe"
  }'
```

### 8.2. Create Teams

```bash
# Create team
awx-cli team create --name DevOps

# Add user to team
awx-cli team associate --team DevOps --user john
```

### 8.3. Configure Permissions

```bash
# Create organization
awx-cli organization create --name MyCompany

# Grant team permissions
awx-cli organization associate --organization MyCompany --team DevOps --role admin

# Create project and assign
awx-cli project create --name MyProject --organization MyCompany --scm-type git --scm-url https://github.com/myorg/ansible-playbooks.git
```

---

## 9. Project Setup

### 9.1. Create Project

```bash
# Using UI
# Navigate to Projects → Add → Create Project
# Fill in details:
# - Name: My Project
# - Organization: MyCompany
# - SCM Type: Git
# - SCM URL: https://github.com/myorg/ansible-playbooks.git
# - Branch: main
# - Sync on Schedule: Enable if needed

# Using CLI
awx-cli project create \
  --name "My Project" \
  --organization MyCompany \
  --scm-type git \
  --scm-url https://github.com/myorg/ansible-playbooks.git \
  --scm-branch main
```

### 9.2. Create Inventory

```bash
# Create inventory
awx-cli inventory create \
  --name "Production Inventory" \
  --organization MyCompany

# Add hosts
awx-cli host create \
  --name web1.example.com \
  --inventory "Production Inventory" \
  --variables '{"ansible_host": "192.168.1.10", "ansible_user": "ansible"}'

# Add groups
awx-cli group create \
  --name webservers \
  --inventory "Production Inventory"

# Add host to group
awx-cli group associate \
  --group webservers \
  --host web1.example.com \
  --inventory "Production Inventory"
```

### 9.3. Create Credentials

```bash
# Create SSH credential
awx-cli credential create \
  --name "SSH Key" \
  --credential-type "Machine" \
  --inputs '{"username": "ansible", "ssh_key_data": "@~/.ssh/id_rsa"}' \
  --organization MyCompany

# Create Vault credential
awx-cli credential create \
  --name "Vault" \
  --credential-type "Vault" \
  --inputs '{"vault_id": "https://vault.example.com", "vault_password": "password"}' \
  --organization MyCompany
```

### 9.4. Create Job Template

```bash
# Create job template
awx-cli job_template create \
  --name "Deploy Application" \
  --project "My Project" \
  --playbook deploy.yml \
  --inventory "Production Inventory" \
  --credential "SSH Key" \
  --organization MyCompany

# Add survey
awx-cli job_template survey create \
  --job-template "Deploy Application" \
  --spec-file survey_spec.json
```

**survey_spec.json example:**
```json
{
  "name": "Deployment Survey",
  "description": "Parameters for deployment",
  "spec": [
    {
      "type": "text",
      "variable": "app_version",
      "question": "Application Version",
      "required": true,
      "default": "latest"
    },
    {
      "type": "multiselect",
      "variable": "environment",
      "question": "Deployment Environment",
      "required": true,
      "choices": ["staging", "production"],
      "default": "staging"
    }
  ]
}
```

---

## 10. Best Practices

### 10.1. Security

1. **Use HTTPS**: Configure SSL/TLS for AWX
2. **RBAC**: Implement proper role-based access control
3. **Credentials**: Use credential types and encryption
4. **Audit Logging**: Enable audit logging for compliance
5. **Regular Updates**: Keep AWX and dependencies updated

### 10.2. Performance

1. **Resource Allocation**: Allocate adequate CPU and memory
2. **Database Tuning**: Optimize PostgreSQL settings
3. **Job Scheduling**: Distribute jobs during off-peak hours
4. **Caching**: Enable caching for improved performance
5. **Monitoring**: Monitor resource usage and job performance

### 10.3. High Availability

1. **Multi-node Deployment**: Deploy AWX in HA mode
2. **Database HA**: Use PostgreSQL HA setup
3. **Load Balancer**: Use external load balancer
4. **Backup Strategy**: Regular backups of database and configuration
5. **Disaster Recovery**: Test disaster recovery procedures

### 10.4. Backup and Recovery

```bash
# Backup AWX database
docker exec awx_postgres pg_dump -U awx awx > awx_backup.sql

# Backup AWX configuration
kubectl get configmap -n awx -o yaml > awx-config-backup.yaml
kubectl get secret -n awx -o yaml > awx-secret-backup.yaml

# Restore database
docker exec -i awx_postgres psql -U awx awx < awx_backup.sql
```

### 10.5. Monitoring

```yaml
# Install Prometheus monitoring
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/devel/config/prometheus/monitoring.yaml

# Configure Grafana dashboards
# Add AWX-specific dashboards for monitoring
```

---

## 11. Troubleshooting

### 11.1. Common Issues

**Issue: AWX UI not accessible**
```bash
# Check pod status
kubectl get pods -n awx

# Check logs
kubectl logs -f deployment/awx-web -n awx

# Check services
kubectl get svc -n awx
```

**Issue: Jobs failing**
```bash
# Check job logs
awx-cli job list --all
awx-cli job get <job-id>

# Check task logs
awx-cli job get <job-id> --type standard
```

**Issue: Database connection errors**
```bash
# Check PostgreSQL pod
kubectl get pods -n awx -l app=postgres

# Check PostgreSQL logs
kubectl logs -f deployment/postgres -n awx

# Test database connection
kubectl exec -it deployment/postgres -n awx -- psql -U awx -d awx
```

### 11.2. Debug Mode

```bash
# Enable debug logging
kubectl set env deployment/awx-web AWX_DEBUG=True -n awx

# Check logs with debug
kubectl logs -f deployment/awx-web -n awx
```

---

## 12. Upgrade

### 12.1. Upgrade AWX

```bash
# Backup before upgrade
./backup_awx.sh

# Upgrade using operator
kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/deploy/awx-operator.yaml

# Upgrade AWX instance
kubectl patch awx awx -n awx --type='json' -p='[{"op": "replace", "path": "/spec/awxVersion", "value":"23.0.0"}]'
```

---

## 13. API Usage

### 13.1. Authentication

```bash
# Get token
curl -X POST http://localhost/api/v2/tokens/ \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "password"}'
```

### 13.2. Common API Calls

```bash
# List jobs
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost/api/v2/jobs/

# Launch job
curl -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost/api/v2/job_templates/1/launch/

# Check job status
curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost/api/v2/jobs/1/
```

---

## References

- [AWX Documentation](https://docs.ansible.com/awx/)
- [AWX Operator Documentation](https://ansible-awx-operator.readthedocs.io/)
- [AWX GitHub Repository](https://github.com/ansible/awx)
- [Ansible Documentation](https://docs.ansible.com/)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
