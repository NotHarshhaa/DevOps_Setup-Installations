# Setup Containerization Tools on Jenkins

This guide provides detailed steps for installing Docker, Docker Compose, Kubernetes CLI (kubectl), and Helm on a Jenkins server.

## Prerequisites

- Ubuntu/Debian or CentOS/RHEL system
- Root or sudo access
- Internet connection
- 64-bit architecture

## 1. Installing Docker

### For Ubuntu/Debian:

```bash
# Update package index
sudo apt update

# Install dependencies
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index
sudo apt update

# Install Docker Engine, Docker CLI, and containerd
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add Jenkins user to docker group
sudo usermod -aG docker jenkins
```

### For CentOS/RHEL:

```bash
# Install dependencies
sudo yum install -y yum-utils

# Add Docker repository
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker Engine
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add Jenkins user to docker group
sudo usermod -aG docker jenkins
```

### Verify Docker Installation:

```bash
docker --version
docker compose version
```

### Test Docker:

```bash
sudo docker run hello-world
```

### Docker Configuration for Jenkins:

```bash
# Create docker group if not exists
sudo groupadd docker

# Add jenkins user to docker group
sudo usermod -aG docker jenkins

# Apply new group membership (logout and login required)
newgrp docker
```

## 2. Installing Docker Compose (Standalone)

**Note**: Docker Compose v2 is included with Docker Engine as a plugin. This is for v1 standalone if needed.

```bash
# Download Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Apply executable permissions
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker-compose --version
```

## 3. Installing Kubernetes CLI (kubectl)

### Using curl (Latest Stable Version):

```bash
# Download kubectl
sudo curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Download checksum
sudo curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

# Validate checksum
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client

# Clean up
rm kubectl kubectl.sha256
```

### Using Package Manager (Ubuntu/Debian):

```bash
# Add Kubernetes signing key
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package index
sudo apt update

# Install kubectl
sudo apt install -y kubectl

# Verify installation
kubectl version --client
```

### Using Package Manager (CentOS/RHEL):

```bash
# Add Kubernetes repository
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
EOF

# Install kubectl
sudo yum install -y kubectl

# Verify installation
kubectl version --client
```

### Configure kubectl for Jenkins:

```bash
# Create .kube directory for Jenkins user
sudo mkdir -p /var/lib/jenkins/.kube
sudo chown -R jenkins:jenkins /var/lib/jenkins/.kube

# Copy kubeconfig file
sudo cp /root/.kube/config /var/lib/jenkins/.kube/config
sudo chown jenkins:jenkins /var/lib/jenkins/.kube/config
sudo chmod 600 /var/lib/jenkins/.kube/config
```

## 4. Installing Helm (Kubernetes Package Manager)

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

### Using Package Manager (Ubuntu/Debian):

```bash
# Add Helm GPG key
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

# Add Helm repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

# Update and install Helm
sudo apt update
sudo apt install -y helm

# Verify installation
helm version
```

## 5. Installing Kind (Kubernetes in Docker) - Optional

For local Kubernetes testing:

```bash
# Download Kind
wget https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

# Make executable
chmod +x kind-linux-amd64

# Move to PATH
sudo mv kind-linux-amd64 /usr/local/bin/kind

# Verify installation
kind version
```

## 6. Installing Minikube - Optional

For local Kubernetes development:

```bash
# Download Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install Minikube
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify installation
minikube version
```

## Jenkins Pipeline Configuration Examples

### Docker Build and Push:

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "myapp:${BUILD_NUMBER}"
        DOCKER_REGISTRY = "registry.example.com"
    }
    
    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build(DOCKER_IMAGE)
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-credentials') {
                        docker.image(DOCKER_IMAGE).push()
                        docker.image(DOCKER_IMAGE).push('latest')
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh 'docker rmi ${DOCKER_IMAGE}'
        }
    }
}
```

### Kubernetes Deployment:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh 'kubectl apply -f k8s/deployment.yaml'
                    sh 'kubectl apply -f k8s/service.yaml'
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh 'kubectl rollout status deployment/myapp -n production'
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
                            --namespace production \
                            --create-namespace
                    '''
                }
            }
        }
    }
}
```

### Docker Compose for Testing:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Start Services') {
            steps {
                sh 'docker compose up -d'
            }
        }
        
        stage('Run Tests') {
            steps {
                sh 'npm test'
            }
        }
        
        stage('Stop Services') {
            steps {
                sh 'docker compose down'
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
        stage('Verify Containerization Tools') {
            steps {
                sh 'docker --version'
                sh 'docker compose version'
                sh 'kubectl version --client'
                sh 'helm version'
            }
        }
    }
}
```

## Best Practices

1. **Use Docker-in-Docker with caution** - prefer Docker socket binding
2. **Secure kubeconfig files** - use Jenkins Credentials
3. **Use specific versions** of kubectl matching your cluster
4. **Implement proper cleanup** in post sections
5. **Use Docker layer caching** for faster builds
6. **Scan Docker images** for vulnerabilities
7. **Use Kubernetes namespaces** for environment isolation
8. **Implement rollback strategies** with Helm

## Troubleshooting

### Docker Permission Issues:

```bash
# Fix docker group permissions
sudo usermod -aG docker jenkins
# Logout and login required
```

### kubectl Connection Issues:

```bash
# Test kubeconfig
kubectl cluster-info --kubeconfig /var/lib/jenkins/.kube/config

# Check permissions
ls -la /var/lib/jenkins/.kube/config
```

### Docker Socket Permission:

```bash
# Add Jenkins user to docker group
sudo usermod -aG docker jenkins

# Or mount docker socket in Jenkins agent
-v /var/run/docker.sock:/var/run/docker.sock
```

### Helm Repository Issues:

```bash
# Update Helm repositories
helm repo update

# List repositories
helm repo list
```

After completing these steps, Docker, Docker Compose, Kubernetes CLI (kubectl), and Helm should be installed and available on your Jenkins server. You can use them in your Jenkins jobs and pipelines to build and deploy Docker containers and interact with Kubernetes clusters.
