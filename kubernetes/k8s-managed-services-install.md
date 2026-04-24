# Setting up Kubernetes Clusters on Managed Services

This guide provides detailed instructions for setting up Kubernetes clusters on managed services including Amazon EKS, Google Kubernetes Engine (GKE), and Azure Kubernetes Service (AKS).

## Amazon EKS (Elastic Kubernetes Service)

### Prerequisites
- An AWS account with appropriate IAM permissions
- AWS CLI v2 installed and configured
- kubectl installed
- eksctl installed (recommended for easier cluster management)

### Option 1: Using eksctl (Recommended)

#### Install eksctl

```bash
# Linux/macOS
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Verify installation
eksctl version
```

#### Create EKS Cluster with eksctl

```bash
# Create a cluster configuration file (eksctl-cluster.yaml)
cat > eksctl-cluster.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks-cluster
  region: us-west-2
  version: "1.29"

managedNodeGroups:
  - name: nodegroup-1
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    volumeSize: 20
    labels:
      role: worker
    tags:
      Environment: production
EOF

# Create the cluster
eksctl create cluster -f eksctl-cluster.yaml
```

#### Quick Cluster Creation

```bash
# Create a simple cluster with default settings
eksctl create cluster --name my-eks-cluster --region us-west-2 --nodes 2
```

#### Configure kubectl

```bash
# eksctl automatically configures kubectl
# Verify connection
kubectl get nodes
```

### Option 2: Using AWS CLI

#### Install AWS IAM Authenticator

```bash
# Linux
curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.6.3/aws-iam-authenticator_0.6.3_linux_amd64
chmod +x ./aws-iam-authenticator
sudo mv ./aws-iam-authenticator /usr/local/bin/aws-iam-authenticator
```

#### Create EKS Cluster using AWS CLI

```bash
# Create VPC and cluster (using eksctl helper or CloudFormation)
eksctl create cluster --name my-eks-cluster --region us-west-2

# Or use AWS CLI directly (requires manual VPC setup)
aws eks create-cluster \
  --name my-eks-cluster \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/eksClusterRole \
  --resources-vpc-config subnetIds=SUBNET_ID1,SUBNET_ID2,securityGroupIds=SECURITY_GROUP_ID \
  --version 1.29
```

#### Update kubeconfig

```bash
# Update kubeconfig with cluster credentials
aws eks update-kubeconfig --name my-eks-cluster --region us-west-2

# Verify connection
kubectl get svc
```

#### Create Node Group

```bash
# Create managed node group
aws eks create-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-node-group \
  --node-role arn:aws:iam::ACCOUNT_ID:role/eksNodeRole \
  --subnets SUBNET_ID1 SUBNET_ID2 \
  --scaling-config minSize=1,maxSize=3,desiredSize=2 \
  --instance-types t3.medium
```

### EKS Additional Features

#### Enable EKS Add-ons

```bash
# Install VPC CNI plugin
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/v1.9/aws-k8s-cni.yaml

# Install AWS Load Balancer Controller
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --set clusterName=my-eks-cluster
```

#### Delete EKS Cluster

```bash
# Using eksctl
eksctl delete cluster --name my-eks-cluster --region us-west-2

# Using AWS CLI
aws eks delete-cluster --name my-eks-cluster --region us-west-2
```

## Google Kubernetes Engine (GKE)

### Prerequisites
- A Google Cloud Platform (GCP) account with appropriate permissions
- Google Cloud SDK (gcloud) installed and configured
- kubectl installed

### Install Google Cloud SDK

```bash
# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
```

### Create GKE Cluster

#### Using gcloud CLI

```bash
# Set your project
gcloud config set project PROJECT_ID

# Create a cluster
gcloud container clusters create my-gke-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=e2-medium \
  --cluster-version=1.29 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5
```

#### Create Autopilot Cluster (Serverless)

```bash
gcloud container clusters create-auto my-autopilot-cluster \
  --region=us-central1
```

### Configure kubectl

```bash
# Get cluster credentials
gcloud container clusters get-credentials my-gke-cluster --zone=us-central1-a

# Verify connection
kubectl get nodes
```

### GKE Additional Features

#### Enable Cloud Monitoring and Logging

```bash
# Enable Google Cloud Operations Suite
gcloud container clusters update my-gke-cluster \
  --zone=us-central1-a \
  --enable-cloud-monitoring \
  --enable-cloud-logging
```

#### Install Network Policies

```bash
# Enable network policy engine
gcloud container clusters update my-gke-cluster \
  --zone=us-central1-a \
  --enable-network-policy

# Install Calico network policy
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/v1.9/aws-k8s-cni.yaml
```

#### Delete GKE Cluster

```bash
gcloud container clusters delete my-gke-cluster --zone=us-central1-a
```

## Azure Kubernetes Service (AKS)

### Prerequisites
- An Azure account with appropriate permissions
- Azure CLI installed and configured
- kubectl installed

### Install Azure CLI

```bash
# Linux
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login to Azure
az login

# Set your subscription
az account set --subscription SUBSCRIPTION_ID
```

### Create AKS Cluster

#### Using Azure CLI

```bash
# Create a resource group
az group create --name myResourceGroup --location eastus

# Create AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 2 \
  --node-vm-size Standard_DS2_v2 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5 \
  --kubernetes-version 1.29 \
  --enable-addons monitoring \
  --generate-ssh-keys
```

#### Create AKS with Advanced Configuration

```bash
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 2 \
  --node-vm-size Standard_DS2_v2 \
  --load-balancer-sku standard \
  --max-pods 110 \
  --network-plugin azure \
  --network-policy azure \
  --enable-private-cluster \
  --enable-managed-identity \
  --generate-ssh-keys
```

### Configure kubectl

```bash
# Get cluster credentials
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Verify connection
kubectl get nodes
```

### AKS Additional Features

#### Enable Azure Container Registry (ACR) Integration

```bash
# Create ACR
az acr create --resource-group myResourceGroup --name myACR --sku Basic

# Attach ACR to AKS
az aks update \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --attach-acr myACR
```

#### Scale the Cluster

```bash
# Scale node pool manually
az aks scale \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --node-count 5

# Enable cluster autoscaler
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10
```

#### Delete AKS Cluster

```bash
# Delete the cluster
az aks delete --resource-group myResourceGroup --name myAKSCluster --yes

# Delete the resource group (deletes all resources)
az group delete --name myResourceGroup --yes
```

## Additional Considerations for All Managed Services

### Networking and Security
- Configure network policies to control pod-to-pod communication
- Set up ingress controllers (NGINX, Traefik, or cloud-native options)
- Configure security groups and firewall rules
- Enable private clusters for enhanced security
- Implement secrets management (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault)

### Scaling and Autoscaling
- Configure Cluster Autoscaler for node-level scaling
- Enable Horizontal Pod Autoscaler (HPA) for pod-level scaling
- Set up Vertical Pod Autoscaler (VPA) for resource optimization
- Use managed node groups for easier scaling

### Monitoring and Logging
- Enable cloud-native monitoring solutions
  - AWS: CloudWatch, X-Ray
  - GCP: Cloud Monitoring, Cloud Logging
  - Azure: Azure Monitor, Log Analytics
- Install Prometheus and Grafana for custom metrics
- Set up centralized logging with ELK stack or similar
- Configure alerting and notifications

### Cost Optimization
- Use spot/preemptible instances for non-critical workloads
- Implement auto-shutdown for development clusters
- Right-size node types based on workload requirements
- Use cost allocation tags for billing analysis

### Backup and Disaster Recovery
- Configure regular backups for persistent volumes
- Implement multi-region replication for critical applications
- Set up disaster recovery procedures
- Use Terraform or similar tools for infrastructure as code

## Comparison Table

| Feature | EKS | GKE | AKS |
|---------|-----|-----|-----|
| **Management Tool** | eksctl / AWS CLI | gcloud CLI | Azure CLI |
| **Autoscaling** | Cluster Autoscaler | Cluster Autoscaler | Cluster Autoscaler |
| **Serverless Option** | Fargate | Autopilot | Virtual Nodes |
| **Monitoring** | CloudWatch | Cloud Monitoring | Azure Monitor |
| **Pricing Model** | Per-hour + EKS fee | Per-hour | Per-hour |
| **Learning Curve** | Medium | Low | Low |

## Official Documentation Links

- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [eksctl Documentation](https://eksctl.io/)
- [Google Kubernetes Engine Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [Azure Kubernetes Service Documentation](https://docs.microsoft.com/azure/aks/)

Always refer to the official documentation for each managed Kubernetes service for the most up-to-date instructions, pricing, and best practices.
