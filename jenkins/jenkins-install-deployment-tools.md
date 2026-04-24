# Setup and Install Deployment Tools on Jenkins

This guide provides detailed steps for installing Ansible, Helm, ArgoCD, and other modern deployment tools on a Jenkins server.

## Prerequisites

- Ubuntu/Debian or CentOS/RHEL system
- Root or sudo access
- Internet connection
- SSH access to target servers (for Ansible)
- Kubernetes cluster access (for Helm/ArgoCD)

## 1. Installing Ansible

### For Ubuntu/Debian:

```bash
# Update package index
sudo apt update

# Install Ansible
sudo apt install -y ansible

# Verify installation
ansible --version
```

### For CentOS/RHEL:

```bash
# Enable EPEL repository
sudo yum install -y epel-release

# Install Ansible
sudo yum install -y ansible

# Verify installation
ansible --version
```

### Using pip (Latest Version):

```bash
# Install Python pip
sudo apt install -y python3-pip

# Install Ansible via pip
pip3 install ansible --user

# Add to PATH
echo 'export PATH=$PATH:$HOME/.local/bin' | sudo tee -a /etc/profile.d/ansible.sh
source /etc/profile.d/ansible.sh

# Verify installation
ansible --version
```

### Configure Ansible for Jenkins:

```bash
# Create Ansible directory for Jenkins
sudo mkdir -p /var/lib/jenkins/ansible
sudo chown -R jenkins:jenkins /var/lib/jenkins/ansible

# Create SSH key for Jenkins user
sudo -u jenkins ssh-keygen -t rsa -b 4096 -f /var/lib/jenkins/.ssh/id_rsa -N ""

# Copy SSH key to target servers (manual step)
# sudo -u jenkins ssh-copy-id user@target-server
```

### Ansible Configuration:

Create `/var/lib/jenkins/ansible/ansible.cfg`:

```ini
[defaults]
inventory = /var/lib/jenkins/ansible/inventory
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
bin_ansible_callbacks = True
```

## 2. Installing Helm (Kubernetes Package Manager)

### Using Script:

```bash
# Download Helm installation script
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

### Manual Installation:

```bash
# Download Helm
wget https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz

# Extract
sudo tar -zxvf helm-v3.14.0-linux-amd64.tar.gz

# Move to PATH
sudo mv linux-amd64/helm /usr/local/bin/helm

# Verify installation
helm version

# Clean up
rm -rf helm-v3.14.0-linux-amd64.tar.gz linux-amd64
```

### Configure Helm for Jenkins:

```bash
# Add Helm repositories
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

## 3. Installing ArgoCD CLI

### Installation:

```bash
# Download ArgoCD CLI
wget https://github.com/argoproj/argo-cd/releases/download/v2.9.0/argocd-linux-amd64

# Make executable
chmod +x argocd-linux-amd64

# Move to PATH
sudo mv argocd-linux-amd64 /usr/local/bin/argocd

# Verify installation
argocd version --client

# Clean up
rm argocd-linux-amd64
```

### Configure ArgoCD:

```bash
# Login to ArgoCD server
argocd login <argocd-server> --auth-token <token>

# Add ArgoCD repository
argocd repo add https://github.com/your-org/your-repo --username <user> --password <token>
```

## 4. Installing kubectl (if not already installed)

See the containerization tools guide for detailed kubectl installation.

## 5. Installing Rsync (for file transfers)

```bash
# For Ubuntu/Debian
sudo apt install -y rsync

# For CentOS/RHEL
sudo yum install -y rsync

# Verify
rsync --version
```

## 6. Installing Terraform (for IaC deployments)

See the IaC tools guide for detailed Terraform installation.

## Jenkins Pipeline Configuration Examples

### Ansible Deployment:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy with Ansible') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'ANSIBLE_KEY')]) {
                    sh '''
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        ansible-playbook -i inventory/production.yml \
                            deploy.yml \
                            --private-key ${ANSIBLE_KEY} \
                            --extra-vars "version=${BUILD_NUMBER}"
                    '''
                }
            }
        }
    }
}
```

### Helm Deployment:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy with Helm') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        helm upgrade --install myapp ./helm-chart \
                            --set image.tag=${BUILD_NUMBER} \
                            --set environment=production \
                            --namespace production \
                            --create-namespace \
                            --wait --timeout 10m
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl rollout status deployment/myapp -n production --timeout=5m
                        kubectl get pods -n production -l app=myapp
                    '''
                }
            }
        }
    }
}
```

### ArgoCD Deployment:

```groovy
pipeline {
    agent any
    
    environment {
        ARGOCD_SERVER = 'argocd.example.com'
        ARGOCD_AUTH_TOKEN = credentials('argocd-token')
    }
    
    stages {
        stage('Deploy with ArgoCD') {
            steps {
                sh '''
                    argocd app sync myapp --server ${ARGOCD_SERVER} --auth-token ${ARGOCD_AUTH_TOKEN}
                    argocd app wait myapp --server ${ARGOCD_SERVER} --auth-token ${ARGOCD_AUTH_TOKEN} --health
                '''
            }
        }
    }
}
```

### Kubernetes Deployment (kubectl):

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl apply -f k8s/configmap.yaml
                        kubectl apply -f k8s/secrets.yaml
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                    '''
                }
            }
        }
        
        stage('Rollout Update') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl set image deployment/myapp \
                            myapp=myapp:${BUILD_NUMBER} \
                            -n production
                    '''
                }
            }
        }
    }
}
```

### Blue-Green Deployment with Ansible:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy to Blue Environment') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh-key', keyFileVariable: 'ANSIBLE_KEY')]) {
                    sh '''
                        ansible-playbook -i inventory/blue.yml \
                            deploy.yml \
                            --private-key ${ANSIBLE_KEY}
                    '''
                }
            }
        }
        
        stage('Health Check Blue') {
            steps {
                sh './scripts/health-check.sh blue'
            }
        }
        
        stage('Switch Traffic') {
            steps {
                sh './scripts/switch-traffic.sh blue'
            }
        }
    }
}
```

### Canary Deployment with Helm:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy Canary') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        helm upgrade --install myapp-canary ./helm-chart \
                            --set image.tag=${BUILD_NUMBER} \
                            --set replicaCount=1 \
                            --namespace production \
                            --values values-canary.yaml
                    '''
                }
            }
        }
        
        stage('Monitor Canary') {
            steps {
                script {
                    sleep(time: 10, unit: 'MINUTES')
                    // Add monitoring logic here
                }
            }
        }
        
        stage('Deploy Full Release') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        helm upgrade --install myapp ./helm-chart \
                            --set image.tag=${BUILD_NUMBER} \
                            --namespace production \
                            --values values-production.yaml
                    '''
                }
            }
        }
    }
}
```

## Verification

Create a test Jenkins job to verify all installations:

```groovy
pipeline {
    agent any
    stages {
        stage('Verify Deployment Tools') {
            steps {
                sh 'ansible --version'
                sh 'helm version'
                sh 'argocd version --client'
                sh 'rsync --version'
            }
        }
    }
}
```

## Best Practices

1. **Use idempotent playbooks** in Ansible for safe repeated deployments
2. **Implement rollback strategies** in all deployment pipelines
3. **Use secrets management** for sensitive data (passwords, keys, tokens)
4. **Implement health checks** after deployments
5. **Use blue-green or canary deployments** for zero-downtime updates
6. **Store configuration in version control** (Ansible playbooks, Helm charts)
7. **Test deployments in staging** before production
8. **Implement proper error handling** and notifications

## Troubleshooting

### Ansible SSH Issues:

```bash
# Test SSH connection
sudo -u jenkins ssh -i /var/lib/jenkins/.ssh/id_rsa user@target-server

# Check Ansible connection
ansible -i inventory/production.yml all -m ping
```

### Helm Chart Issues:

```bash
# Lint Helm chart
helm lint ./helm-chart

# Dry run deployment
helm upgrade --install myapp ./helm-chart --dry-run --debug
```

### Kubernetes Connection Issues:

```bash
# Test kubeconfig
kubectl cluster-info --kubeconfig /var/lib/jenkins/.kube/config

# Check permissions
kubectl auth can-i get pods --as=jenkins
```

### Permission Issues:

```bash
# Fix permissions for Jenkins user
sudo chown -R jenkins:jenkins /var/lib/jenkins/ansible
sudo chown -R jenkins:jenkins /var/lib/jenkins/.ssh
```

Once Ansible, Helm, ArgoCD, and other deployment tools are installed on your Jenkins server, you can use them in your Jenkins jobs or pipelines to automate deployment tasks. You can invoke Ansible playbooks, Helm charts, or ArgoCD commands directly from your Jenkinsfile or from shell commands within your Jenkins jobs. Make sure to configure any necessary credentials or configuration files required by these tools to interact with your deployment targets.
