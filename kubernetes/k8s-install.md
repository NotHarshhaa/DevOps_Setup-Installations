# Kubernetes (K8S) step-by-step installation guide on Ubuntu from scratch

![k8s](https://imgur.com/bKUeyKX.png)

This guide will walk you through the process of setting up a basic Kubernetes cluster using `kubeadm`, `kubelet`, and `kubectl` with containerd as the container runtime.

## Prerequisites

1. **Ubuntu**: You should have a clean installation of Ubuntu (20.04 LTS or 22.04 LTS recommended) on the machines where you plan to set up your Kubernetes cluster.

2. **Minimum 2 Nodes**: For a basic Kubernetes cluster, you need at least two nodes - one for the control plane and one or more for worker nodes.

3. **Network Configuration**: Ensure that your nodes can communicate with each other over the network. You should have a static IP address for each node.

4. **Root Privileges**: You need to run some commands with root privileges, so make sure you have `sudo` access.

5. **Hardware Requirements**:
   - 2 GB RAM or more per machine (2GB+ recommended for control plane)
   - 2 CPUs or more
   - Stable internet connection for pulling images

## Step 1: Update System and Install Required Packages

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

## Step 2: Install and Configure containerd

Kubernetes uses containerd as the default container runtime since v1.24.

```bash
# Add Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Set up the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt update
sudo apt install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup for better compatibility with Kubernetes
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Start and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Step 3: Disable Swap

Kubernetes requires that swap be disabled.

```bash
# Disable swap temporarily
sudo swapoff -a

# Disable swap permanently by commenting out swap entry in /etc/fstab
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Step 4: Load Kernel Modules and Configure System Parameters

```bash
# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters
sudo sysctl --system
```

## Step 5: Install Kubernetes Tools

```bash
# Add Kubernetes apt repository GPG key
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list and install Kubernetes components
sudo apt update
sudo apt install -y kubelet kubeadm kubectl

# Pin Kubernetes package version to prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl
```

## Step 6: Initialize the Kubernetes Control Plane Node

On your control plane node, initialize the Kubernetes cluster using `kubeadm`. Replace `CONTROL_PLANE_IP` with the actual IP address of your control plane node.

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=CONTROL_PLANE_IP
```

After the initialization is complete, you will see a message with a `kubeadm join` command to add worker nodes to the cluster. **Save this command for later use.**

## Step 7: Set Up kubectl for Your User

Run these commands on your control plane node to configure `kubectl` for your user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 8: Install a Network Plugin (CNI)

Choose a network plugin for your cluster. Calico is recommended for production environments.

```bash
# Install Calico CNI
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Alternative: Flannel (simpler, suitable for testing)
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

## Step 9: Join Worker Nodes

On each worker node, run the `kubeadm join` command you saved from Step 6. It should look something like this:

```bash
sudo kubeadm join CONTROL_PLANE_IP:6443 --token TOKEN --discovery-token-ca-cert-hash SHA256_HASH
```

If you've lost the join token, you can generate a new one on the control plane:
```bash
kubeadm token create --print-join-command
```

## Step 10: Verify Cluster Status

Back on the control plane node, check the status of your cluster:

```bash
# Check node status
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check cluster info
kubectl cluster-info
```

You should see all nodes in the "Ready" state and all system pods running.

## Optional: Enable Autocompletion for kubectl

```bash
# Bash autocompletion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc

# Zsh autocompletion
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
source ~/.zshrc
```

## Troubleshooting

**If nodes are NotReady:**
```bash
# Check kubelet logs
sudo journalctl -xeu kubelet

# Check if CNI pods are running
kubectl get pods -n kube-system
```

**If kubeadm init fails:**
```bash
# Reset and retry
sudo kubeadm reset
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Congratulations! You've successfully installed Kubernetes on Ubuntu from scratch. You can now deploy and manage containers using Kubernetes.

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
