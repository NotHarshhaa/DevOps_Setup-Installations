# Install and Set Up Minikube on Ubuntu and CentOS

![](https://imgur.com/rl09sqv.png)

This guide will help you install and set up Minikube on both Ubuntu and CentOS/RHEL systems. Minikube runs a single-node Kubernetes cluster on your local machine, making it ideal for development and testing.

## Prerequisites

### System Requirements
- 2 CPUs or more
- 2GB of free memory
- 20GB of free disk space
- Internet connection
- Container or virtual machine manager (Docker, Podman, VirtualBox, KVM, etc.)

### Supported Drivers
Minikube supports multiple drivers for running the Kubernetes cluster:
- **docker** (recommended for most users)
- **podman**
- **virtualbox**
- **kvm2** (KVM)
- **vmware**
- **ssh**
- **none** (runs directly on the host, requires additional setup)

## Installation for Ubuntu/Debian

### Step 1: Install Dependencies

```bash
sudo apt-get update
sudo apt-get install -y curl wget apt-transport-https
```

### Step 2: Install Docker (Recommended Driver)

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to the docker group
sudo usermod -aG docker $USER

# Log out and log back in for group changes to take effect
```

Alternative: Install VirtualBox (if preferred)
```bash
sudo apt-get install -y virtualbox virtualbox-ext-pack
```

### Step 3: Install kubectl

```bash
# Download the latest release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
```

### Step 4: Install Minikube

```bash
# Download Minikube binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install Minikube
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify installation
minikube version
```

## Installation for CentOS/RHEL

### Step 1: Install Dependencies

```bash
sudo yum update -y
sudo yum install -y curl wget
```

### Step 2: Install Docker (Recommended Driver)

```bash
# Install Docker
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to the docker group
sudo usermod -aG docker $USER

# Log out and log back in for group changes to take effect
```

Alternative: Install VirtualBox (if preferred)
```bash
sudo yum install -y epel-release
sudo yum install -y VirtualBox
```

### Step 3: Install kubectl

```bash
# Download the latest release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
```

### Step 4: Install Minikube

```bash
# Download Minikube binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install Minikube
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify installation
minikube version
```

## Starting Minikube

### Start with Docker Driver (Recommended)

```bash
minikube start --driver=docker
```

### Start with Specific Kubernetes Version

```bash
minikube start --driver=docker --kubernetes-version=v1.29.0
```

### Start with Custom Resources

```bash
minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=50g
```

### Start with VirtualBox Driver

```bash
minikube start --driver=virtualbox
```

## Verify Installation

```bash
# Check Minikube status
minikube status

# Check cluster info
kubectl cluster-info

# Check nodes
kubectl get nodes

# Check all pods
kubectl get pods -A
```

## Accessing Kubernetes Dashboard

### Enable Dashboard

```bash
minikube dashboard
```

This will automatically open the dashboard in your default browser.

### Access Dashboard Manually

```bash
# Start the dashboard in background
minikube dashboard &

# Or use kubectl proxy
kubectl proxy
```

Then access at: `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

### Get Dashboard Token

```bash
# Create a service account and get token
kubectl -n kubernetes-dashboard create token admin-user

# Or describe the secret (older method)
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

## Common Minikube Commands

```bash
# Start Minikube
minikube start

# Stop Minikube
minikube stop

# Delete Minikube cluster
minikube delete

# View Minikube logs
minikube logs

# Open Minikube dashboard
minikube dashboard

# SSH into Minikube VM
minikube ssh

# Get Minikube IP
minikube ip

# Mount a directory into Minikube
minikube mount <source>:<target>

# List addons
minikube addons list

# Enable an addon
minikube addons enable <addon-name>

# Disable an addon
minikube addons disable <addon-name>
```

## Useful Addons

Minikube comes with several useful addons:

```bash
# Enable ingress controller
minikube addons enable ingress

# Enable metrics server
minikube addons enable metrics-server

# Enable dashboard
minikube addons enable dashboard

# Enable registry
minikube addons enable registry
```

## Troubleshooting

### Minikube fails to start

```bash
# Check Minikube status
minikube status

# Delete and restart
minikube delete
minikube start --driver=docker
```

### Permission denied errors

```bash
# Ensure your user is in the docker group
sudo usermod -aG docker $USER
# Log out and log back in
```

### Out of memory errors

```bash
# Start with more memory
minikube start --driver=docker --memory=8192
```

### Driver issues

```bash
# Check available drivers
minikube config view

# Switch drivers
minikube config set driver docker
minikube delete
minikube start
```

## Uninstalling Minikube

```bash
# Stop and delete Minikube
minikube stop
minikube delete

# Remove Minikube binary
sudo rm /usr/local/bin/minikube

# Remove kubectl (optional)
sudo rm /usr/local/bin/kubectl

# Remove Minikube configuration
rm -rf ~/.minikube
```

## Additional Resources

- [Official Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [Minikube Drivers](https://minikube.sigs.k8s.io/docs/drivers/)
- [Minikube Addons](https://minikube.sigs.k8s.io/docs/handbook/addons/)

**Note:** Minikube is primarily designed for development and testing purposes. For production workloads, consider using a full Kubernetes cluster setup or managed Kubernetes services.
