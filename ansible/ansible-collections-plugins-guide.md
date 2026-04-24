# Ansible Collections and Plugins Guide

![Ansible Collections](https://www.ansible.com/hubfs/2019_Assets/Red_Hat_Ansible_Automation_Platform-Logo-RGB_White.png)

Comprehensive guide for using, creating, and managing Ansible collections and plugins for extended automation capabilities.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Popular Collections](#2-popular-collections)
3. [AWS Collection](#3-aws-collection)
4. [Kubernetes Collection](#4-kubernetes-collection)
5. [Docker Collection](#6-docker-collection)
6. [Community Collections](#7-community-collections)
7. [Custom Collections](#8-custom-collections)
8. [Custom Plugins](#9-custom-plugins)
9. [Best Practices](#10-best-practices)

---

## 1. Introduction

### 1.1. What are Ansible Collections?

Ansible collections are a distribution format for Ansible content that can include playbooks, roles, modules, plugins, and documentation. They provide a way to package and distribute automation content.

### 1.2. Benefits of Collections
- **Modularization**: Organize related content
- **Versioning**: Independent version management
- **Distribution**: Easy sharing and reuse
- **Namespacing**: Avoid naming conflicts
- **Documentation**: Integrated documentation

### 1.3. Collection Structure

```plaintext
namespace.collectionname/
├── docs/
├── plugins/
│   ├── modules/
│   ├── inventory/
│   ├── lookup/
│   ├── filter/
│   └── callback/
├── roles/
├── playbooks/
├── tests/
├── galaxy.yml
├── README.md
└── LICENSE
```

---

## 2. Popular Collections

### 2.1. Essential Collections

```bash
# Install essential collections
ansible-galaxy collection install community.general
ansible-galaxy collection install community.docker
ansible-galaxy collection install community.postgresql
ansible-galaxy collection install community.mysql
ansible-galaxy collection install community.crypto
ansible-galaxy collection install community.windows
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install google.cloud
ansible-galaxy collection install azure.azcollection
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install ansible.posix
ansible-galaxy collection install ansible.windows
```

### 2.2. Requirements File

```yaml
# requirements.yml
collections:
  - name: community.general
    version: ">=6.0.0"
  - name: community.docker
    version: ">=3.0.0"
  - name: amazon.aws
    version: ">=5.0.0"
  - name: kubernetes.core
    version: ">=2.0.0"
  - name: ansible.posix
    version: ">=1.0.0"

roles:
  - name: geerlingguy.nginx
    version: "3.2.0"
  - name: geerlingguy.mysql
    version: "4.3.0"
```

```bash
# Install from requirements file
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml
```

---

## 3. AWS Collection

### 3.1. Installation

```bash
# Install AWS collection
ansible-galaxy collection install amazon.aws

# Install boto3 and botocore
pip install boto3 botocore

# Configure AWS credentials
aws configure
```

### 3.2. EC2 Management

```yaml
# Create EC2 instance
- name: Launch EC2 instance
  amazon.aws.ec2_instance:
    name: my-instance
    instance_type: t3.micro
    image_id: ami-0c55b159cbfafe1f0
    region: us-west-2
    vpc_subnet_id: subnet-12345
    security_groups:
      - sg-12345
    key_name: my-key-pair
    tags:
      Environment: production
      Application: myapp
    state: present
  register: ec2_instance

- name: Wait for instance to be running
  amazon.aws.ec2_instance_info:
    instance_ids:
      - "{{ ec2_instance.instance_ids[0] }}"
    region: us-west-2
  register: ec2_info
  until: ec2_info.instances[0].state.name == 'running'
  retries: 10
  delay: 30
```

### 3.3. S3 Management

```yaml
# Create S3 bucket
- name: Create S3 bucket
  amazon.aws.s3_bucket:
    name: my-bucket
    region: us-west-2
    state: present
    public_access:
      block_public_acls: true
      block_public_policy: true
    tags:
      Environment: production

# Upload file to S3
- name: Upload file to S3
  amazon.aws.s3_object:
    bucket: my-bucket
    object: /path/to/file.txt
    src: /local/file.txt
    mode: put

# Sync directory to S3
- name: Sync directory to S3
  community.aws.s3_sync:
    bucket: my-bucket
    path: /local/directory/
    delete: true
```

### 3.4. RDS Management

```yaml
# Create RDS instance
- name: Create RDS instance
  amazon.aws.rds_instance:
    db_instance_identifier: mydb
    db_instance_class: db.t3.micro
    engine: postgres
    engine_version: "14.7"
    master_username: admin
    master_user_password: "{{ vault_db_password }}"
    allocated_storage: 20
    storage_type: gp2
    region: us-west-2
    vpc_security_group_ids:
      - sg-12345
    db_subnet_group_name: my-subnet-group
    tags:
      Environment: production
    state: present
```

### 3.5. Lambda Management

```yaml
# Deploy Lambda function
- name: Deploy Lambda function
  amazon.aws.lambda:
    name: my-function
    state: present
    runtime: python3.9
    handler: index.handler
    zip_file: /path/to/function.zip
    role: arn:aws:iam::123456789012:role/lambda-role
    region: us-west-2
    environment_variables:
      ENV: production
      DEBUG: "false"
```

---

## 4. Kubernetes Collection

### 4.1. Installation

```bash
# Install Kubernetes collection
ansible-galaxy collection install kubernetes.core

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Python Kubernetes library
pip install kubernetes
```

### 4.2. Deploy Resources

```yaml
# Create namespace
- name: Create namespace
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: my-namespace

# Create deployment
- name: Create deployment
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
        namespace: my-namespace
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
              resources:
                requests:
                  cpu: 250m
                  memory: 256Mi
                limits:
                  cpu: 500m
                  memory: 512Mi
```

### 4.3. Helm Chart Management

```yaml
# Add Helm repository
- name: Add Helm repository
  kubernetes.core.helm_repository:
    name: stable
    repo_url: https://charts.helm.sh/stable

# Install Helm chart
- name: Install Helm chart
  kubernetes.core.helm:
    name: nginx
    chart_ref: stable/nginx-ingress
    release_namespace: ingress-nginx
    values:
      controller:
        service:
          type: LoadBalancer
```

### 4.4. ConfigMaps and Secrets

```yaml
# Create ConfigMap
- name: Create ConfigMap
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: app-config
        namespace: my-namespace
      data:
        app.properties: |
          server.port=8080
          database.url=jdbc:postgresql://localhost:5432/mydb

# Create Secret
- name: Create Secret
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Secret
      metadata:
        name: app-secret
        namespace: my-namespace
      type: Opaque
      stringData:
        password: "{{ vault_app_password }}"
        api-key: "{{ vault_api_key }}"
```

---

## 5. Docker Collection

### 5.1. Installation

```bash
# Install Docker collection
ansible-galaxy collection install community.docker

# Install Docker SDK for Python
pip install docker
```

### 5.2. Container Management

```yaml
# Pull Docker image
- name: Pull Docker image
  community.docker.docker_image:
    name: nginx:latest
    source: pull
    state: present

# Run Docker container
- name: Run Docker container
  community.docker.docker_container:
    name: mycontainer
    image: nginx:latest
    state: started
    ports:
      - "80:80"
    volumes:
      - /data:/data
    env:
      ENV: production
    restart_policy: unless-stopped

# Stop container
- name: Stop container
  community.docker.docker_container:
    name: mycontainer
    state: stopped
```

### 5.3. Image Management

```yaml
# Build Docker image
- name: Build Docker image
  community.docker.docker_image:
    name: myapp:latest
    build:
      path: /path/to/dockerfile
      pull: true
    source: build
    state: present

# Tag and push image
- name: Tag and push image
  community.docker.docker_image:
    name: registry.example.com/myapp:latest
    repository: myapp
    tag: latest
    push: true
    source: local
```

### 5.4. Docker Network Management

```yaml
# Create Docker network
- name: Create Docker network
  community.docker.docker_network:
    name: my-network
    driver: bridge
    state: present

# Connect container to network
- name: Connect container to network
  community.docker.docker_network:
    name: my-network
    connected:
      - mycontainer
    state: present
```

---

## 6. Community Collections

### 6.1. PostgreSQL Collection

```yaml
# Install collection
ansible-galaxy collection install community.postgresql

# Create database
- name: Create PostgreSQL database
  community.postgresql.postgresql_db:
    name: mydb
    state: present

# Create user
- name: Create PostgreSQL user
  community.postgresql.postgresql_user:
    db: mydb
    name: myuser
    password: "{{ vault_db_password }}"
    priv: "ALL"
    state: present
```

### 6.2. MySQL Collection

```yaml
# Install collection
ansible-galaxy collection install community.mysql

# Create database
- name: Create MySQL database
  community.mysql.mysql_db:
    name: mydb
    state: present

# Create user
- name: Create MySQL user
  community.mysql.mysql_user:
    name: myuser
    password: "{{ vault_db_password }}"
    priv: "mydb.*:ALL"
    state: present
```

### 6.3. Crypto Collection

```yaml
# Install collection
ansible-galaxy collection install community.crypto

# Generate SSH key
- name: Generate SSH key pair
  community.crypto.openssh_keypair:
    path: ~/.ssh/my_key
    type: ed25519

# Encrypt file
- name: Encrypt file
  community.crypto.openssl_encrypt:
    content: "{{ sensitive_data }}"
    dest: /path/to/encrypted.enc
    passphrase: "{{ vault_passphrase }}"
```

---

## 7. Custom Collections

### 7.1. Create Collection Structure

```bash
# Initialize collection
ansible-galaxy collection init mynamespace.mycollection

# Directory structure
cd mynamespace.mycollection
ls -la
# plugins/
# roles/
# playbooks/
# tests/
# docs/
# galaxy.yml
```

### 7.2. galaxy.yml Configuration

```yaml
# galaxy.yml
namespace: mynamespace
name: mycollection
version: 1.0.0
authors:
  - Your Name <your.email@example.com>
description: My custom Ansible collection
license: MIT
tags:
  - collection
  - automation
dependencies:
  community.general: ">=6.0.0"
repository: https://github.com/your-org/mynamespace.mycollection
documentation: https://github.com/your-org/mynamespace.mycollection/blob/main/README.md
homepage: https://github.com/your-org/mynamespace.mycollection
issues: https://github.com/your-org/mynamespace.mycollection/issues
build_ignore:
  - .gitignore
  - .github
```

### 7.3. Create Custom Module

```python
# plugins/modules/my_module.py
#!/usr/bin/python
from __future__ import absolute_import, division, print_function
__metaclass__ = type

from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(
            name=dict(type='str', required=True),
            state=dict(type='str', default='present', choices=['present', 'absent']),
        ),
        supports_check_mode=True
    )

    name = module.params['name']
    state = module.params['state']

    if module.check_mode:
        module.exit_json(changed=False)

    # Your module logic here
    changed = True
    result = {
        'name': name,
        'state': state,
        'changed': changed
    }

    module.exit_json(**result)

if __name__ == '__main__':
    main()
```

### 7.4. Create Custom Filter Plugin

```python
# plugins/filter/my_filters.py
def reverse_string(s):
    return s[::-1]

def to_upper(s):
    return s.upper()

class FilterModule(object):
    def filters(self):
        return {
            'reverse_string': reverse_string,
            'to_upper': to_upper
        }
```

### 7.5. Build and Publish Collection

```bash
# Build collection
ansible-galaxy collection build

# Install locally
ansible-galaxy collection install mynamespace-mycollection-1.0.0.tar.gz

# Publish to Galaxy
ansible-galaxy collection publish mynamespace-mycollection-1.0.0.tar.gz --api-key YOUR_GALAXY_API_KEY

# Publish to private repository
ansible-galaxy collection publish mynamespace-mycollection-1.0.0.tar.gz --server https://galaxy.example.com --api-key YOUR_API_KEY
```

---

## 8. Custom Plugins

### 8.1. Lookup Plugins

```python
# plugins/lookup/my_lookup.py
from __future__ import absolute_import, division, print_function
__metaclass__ = type

from ansible.plugins.lookup import LookupBase
from ansible.errors import AnsibleError

DOCUMENTATION = """
    lookup: my_lookup
    description: Custom lookup plugin
    options:
      _terms:
        description: Terms to lookup
        required: True
"""

EXAMPLES = """
    - name: Use custom lookup
      debug:
        msg: "{{ lookup('my_lookup', 'term1', 'term2') }}"
"""

class LookupModule(LookupBase):
    def run(self, terms, variables=None, **kwargs):
        result = []
        for term in terms:
            # Your lookup logic here
            result.append(f"Processed: {term}")
        return result
```

### 8.2. Inventory Plugins

```python
# plugins/inventory/my_inventory.py
from __future__ import absolute_import, division, print_function
__metaclass__ = type

from ansible.plugins.inventory import BaseInventoryPlugin
from ansible.errors import AnsibleError
import yaml

DOCUMENTATION = """
    name: my_inventory
    plugin_type: inventory
    description: Custom inventory plugin
    options:
      config_file:
        description: Path to config file
        required: True
"""

class InventoryModule(BaseInventoryPlugin):
    NAME = 'my_inventory'

    def verify_file(self, path):
        return path.endswith('.yml') or path.endswith('.yaml')

    def parse(self, inventory, loader, path, cache=True):
        super(InventoryModule, self).parse(inventory, loader, path, cache)
        
        config = self._read_config_file(path)
        
        for host in config.get('hosts', []):
            self.inventory.add_host(host['name'])
            self.inventory.set_variable(host['name'], 'ansible_host', host['ip'])

    def _read_config_file(self, path):
        with open(path) as f:
            return yaml.safe_load(f)
```

### 8.3. Callback Plugins

```python
# plugins/callback/my_callback.py
from __future__ import absolute_import, division, print_function
__metaclass__ = type

from ansible.plugins.callback import CallbackBase

class CallbackModule(CallbackBase):
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'notification'
    CALLBACK_NAME = 'my_callback'

    def v2_playbook_on_task_start(self, task, is_conditional):
        self._display.display(f"Starting task: {task.name}", color='blue')

    def v2_playbook_on_stats(self, stats):
        self._display.display("Playbook completed", color='green')
```

---

## 9. Best Practices

### 9.1. Collection Usage

1. **Use FQCN**: Always use fully qualified collection names
2. **Version Pinning**: Pin specific versions in requirements.yml
3. **Documentation**: Document collection usage and dependencies
4. **Testing**: Test collection usage in isolated environments
5. **Updates**: Regularly update collections for security patches

### 9.2. Custom Collections

1. **Namespace**: Use appropriate namespace for your organization
2. **Versioning**: Follow semantic versioning
3. **Testing**: Include comprehensive tests
4. **Documentation**: Provide clear documentation
5. **Dependencies**: Minimize external dependencies

### 9.3. Plugin Development

1. **Idempotency**: Ensure plugins are idempotent
2. **Error Handling**: Implement proper error handling
3. **Documentation**: Add DOCUMENTATION and EXAMPLES
4. **Testing**: Write unit tests for plugins
5. **Performance**: Optimize for performance

### 9.4. Security

1. **Validation**: Validate all inputs
2. **Sanitization**: Sanitize user-provided data
3. **Secrets**: Never log sensitive information
4. **Permissions**: Follow principle of least privilege
5. **Auditing**: Log plugin usage for audit trails

---

## 10. Examples

### 10.1. Complete AWS Deployment

```yaml
# deploy_aws_infrastructure.yml
---
- name: Deploy AWS Infrastructure
  hosts: localhost
  gather_facts: no
  vars:
    region: us-west-2
    vpc_cidr: 10.0.0.0/16
    subnet_cidr: 10.0.1.0/24
    
  tasks:
    - name: Create VPC
      amazon.aws.ec2_vpc_net:
        name: my-vpc
        cidr_block: "{{ vpc_cidr }}"
        region: "{{ region }}"
        state: present
      register: vpc

    - name: Create subnet
      amazon.aws.ec2_vpc_subnet:
        vpc_id: "{{ vpc.vpc.id }}"
        cidr: "{{ subnet_cidr }}"
        region: "{{ region }}"
        state: present
      register: subnet

    - name: Create security group
      amazon.aws.ec2_security_group:
        name: my-sg
        description: My security group
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ region }}"
        rules:
          - proto: tcp
            ports:
              - 80
              - 443
            cidr: 0.0.0.0/0
        state: present
      register: sg

    - name: Launch EC2 instance
      amazon.aws.ec2_instance:
        name: my-instance
        instance_type: t3.micro
        image_id: ami-0c55b159cbfafe1f0
        region: "{{ region }}"
        vpc_subnet_id: "{{ subnet.subnet.id }}"
        security_groups:
          - "{{ sg.group_id }}"
        state: present
```

### 10.2. Kubernetes Application Deployment

```yaml
# deploy_k8s_app.yml
---
- name: Deploy Application to Kubernetes
  hosts: localhost
  gather_facts: no
  vars:
    namespace: production
    app_name: myapp
    app_image: myapp:latest
    replicas: 3
    
  tasks:
    - name: Create namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ namespace }}"

    - name: Create ConfigMap
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: ConfigMap
          metadata:
            name: "{{ app_name }}-config"
            namespace: "{{ namespace }}"
          data:
            config.yaml: |
              server.port: 8080
              log.level: info

    - name: Create Secret
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Secret
          metadata:
            name: "{{ app_name }}-secret"
            namespace: "{{ namespace }}"
          type: Opaque
          stringData:
            database-password: "{{ vault_db_password }}"

    - name: Create Deployment
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: "{{ app_name }}"
            namespace: "{{ namespace }}"
          spec:
            replicas: "{{ replicas }}"
            selector:
              matchLabels:
                app: "{{ app_name }}"
            template:
              metadata:
                labels:
                  app: "{{ app_name }}"
              spec:
                containers:
                - name: "{{ app_name }}"
                  image: "{{ app_image }}"
                  ports:
                  - containerPort: 8080
                  env:
                  - name: CONFIG_PATH
                    value: /etc/config/config.yaml
                  volumeMounts:
                  - name: config
                    mountPath: /etc/config
                  - name: secret
                    mountPath: /etc/secrets
                  volumes:
                  - name: config
                    configMap:
                      name: "{{ app_name }}-config"
                  - name: secret
                    secret:
                      secretName: "{{ app_name }}-secret"

    - name: Create Service
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Service
          metadata:
            name: "{{ app_name }}-service"
            namespace: "{{ namespace }}"
          spec:
            selector:
              app: "{{ app_name }}"
            ports:
            - port: 80
              targetPort: 8080
            type: LoadBalancer
```

---

## References

- [Ansible Collections Documentation](https://docs.ansible.com/ansible/latest/collections/index.html)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Ansible Plugin Development](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections.html)
- [Collection Naming Requirements](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections.html#collection-requirements)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
