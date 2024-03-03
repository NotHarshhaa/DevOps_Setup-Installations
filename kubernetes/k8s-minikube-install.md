# To Install and set up Minikube on both Ubuntu and CentOS

![](https://imgur.com/rl09sqv.png)

To set up Minikube on both Ubuntu and CentOS, you'll need to follow similar steps, but there might be some minor differences in package installation or configurations. Here's a general guide for both:

### Prerequisites:

- **Hypervisor**: Minikube requires a hypervisor to run the Kubernetes clusters. You can choose from VirtualBox, KVM, VMware, etc. Install one according to your preference.
- **kubectl**: This is the Kubernetes command-line tool. You can install it using the package manager of your distribution.

### Steps to Install Minikube:

#### 1. Install Dependencies:

**Ubuntu:**
```bash
sudo apt-get update
sudo apt-get install apt-transport-https curl
```

**CentOS:**
```bash
sudo yum install -y curl
```

#### 2. Install kubectl:

```bash
# Ubuntu
sudo snap install kubectl --classic

# CentOS
sudo yum install -y kubectl
```

#### 3. Install Minikube:

```bash
# Ubuntu
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# CentOS
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

#### 4. Start Minikube:

```bash
minikube start --driver=<driver-name>
```
Replace `<driver-name>` with the name of the hypervisor you installed (e.g., virtualbox, kvm2, etc.).

#### 5. Verify Installation:

```bash
minikube status
kubectl cluster-info
kubectl get nodes
```

This should show you the status of your Minikube cluster and the nodes.

#### 6. (Optional) Install Dashboard:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

#### 7. Access Kubernetes Dashboard:

```bash
kubectl proxy
```

You can access the dashboard at `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/` with the token obtained using:

```bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

**Note:** Minikube runs a single-node Kubernetes cluster on your machine, and it's mainly meant for development and testing purposes.

That's it! You should now have Minikube up and running on both Ubuntu and CentOS environments.
