# AWS EKS Cluster Setup and Management Guide

![banner](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2023/11/16/Workflow.png)

## 1. Prerequisites

### 1.1. Required Tools
- AWS CLI v2.0+
- kubectl
- eksctl
- AWS IAM Authenticator
- Helm v3

### 1.2. AWS Configuration
```bash
# Configure AWS CLI
aws configure

# Verify configuration
aws sts get-caller-identity
```

## 2. EKS Cluster Creation

### 2.1. Using eksctl (Recommended Method)

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production-eks
  region: us-west-2
  version: "1.24"

managedNodeGroups:
  - name: managed-nodes
    instanceType: t3.large
    minSize: 2
    maxSize: 5
    desiredCapacity: 3
    volumeSize: 100
    ssh:
      allow: true
      publicKeyName: your-key-name
    labels:
      role: worker
    tags:
      Environment: production
    iam:
      withAddonPolicies:
        autoScaler: true
        albIngress: true
        cloudWatch: true

vpc:
  cidr: "10.0.0.0/16"
  nat:
    gateway: Single
  clusterEndpoints:
    publicAccess: true
    privateAccess: true
```

```bash
# Create cluster
eksctl create cluster -f cluster-config.yaml

# Verify cluster creation
aws eks describe-cluster --name production-eks --region us-west-2
```

### 2.2. Using AWS Console (Alternative Method)
1. Open AWS Console
2. Navigate to EKS service
3. Click "Create cluster"
4. Fill in required details:
   - Cluster name
   - Kubernetes version
   - Role selection
   - VPC configuration
   - Security group settings
   - Logging preferences

## 3. Cluster Access Configuration

### 3.1. Configure kubectl

```bash
# Update kubeconfig
aws eks update-kubeconfig --name production-eks --region us-west-2

# Verify connection
kubectl get nodes
```

### 3.2. IAM Authentication

```bash
# Install AWS IAM Authenticator
curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/aws-iam-authenticator
chmod +x ./aws-iam-authenticator
sudo mv aws-iam-authenticator /usr/local/bin

# Add IAM users/roles
eksctl create iamidentitymapping \
    --cluster production-eks \
    --arn arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME \
    --username admin \
    --group system:masters
```

## 4. Essential Add-ons Installation

### 4.1. AWS Load Balancer Controller

```bash
# Add Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=production-eks \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 4.2. Cluster Autoscaler

```yaml
# cluster-autoscaler.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/AWSEKSClusterAutoscalerRole
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.24.0
          name: cluster-autoscaler
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --skip-nodes-with-system-pods=false
            - --balance-similar-node-groups
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/production-eks
```

### 4.3. AWS CloudWatch Container Insights

```bash
# Enable Container Insights
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest.yaml
```

## 5. Networking Configuration

### 5.1. VPC CNI Settings

```bash
# Update CNI version
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.12/config/master/aws-k8s-cni.yaml

# Configure CNI
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
```

### 5.2. Network Policies

```yaml
# default-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## 6. Security Configuration

### 6.1. Pod Security Policies

```yaml
# restricted-psp.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
```

### 6.2. RBAC Configuration

```yaml
# rbac-config.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developer-binding
subjects:
- kind: Group
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

## 7. Monitoring and Logging

### 7.1. CloudWatch Logs

```bash
# Enable CloudWatch logging
eksctl utils update-cluster-logging \
    --enable-types all \
    --cluster production-eks \
    --region us-west-2
```

### 7.2. Prometheus and Grafana Setup

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace
```

## 8. Cost Optimization

### 8.1. Resource Quotas

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

### 8.2. Spot Instances Configuration

```yaml
# spot-nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-eks
  region: us-west-2

managedNodeGroups:
  - name: spot-nodes
    instanceTypes: ["t3.large", "t3a.large", "t3.xlarge"]
    spot: true
    minSize: 2
    maxSize: 5
    desiredCapacity: 3
```

## 9. Backup and Disaster Recovery

### 9.1. Velero Setup

```bash
# Install Velero
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.4.0 \
    --bucket backup-bucket \
    --backup-location-config region=us-west-2 \
    --snapshot-location-config region=us-west-2 \
    --secret-file ./credentials-velero
```

### 9.2. Regular Backup Configuration

```yaml
# scheduled-backup.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
spec:
  schedule: "0 1 * * *"
  template:
    includedNamespaces:
    - default
    - kube-system
    ttl: 720h
```

## 10. Best Practices

### 10.1. Cluster Management
- Regularly update EKS version
- Use managed node groups
- Implement proper tagging strategy
- Regular security patches
- Monitor cluster health

### 10.2. Security
- Enable control plane logging
- Use security groups effectively
- Implement pod security policies
- Regular IAM review
- Enable encryption at rest

### 10.3. Networking
- Use private endpoints where possible
- Implement network policies
- Configure proper VPC settings
- Use service mesh for complex applications
- Regular network security review

### 10.4. Cost Management
- Use spot instances where appropriate
- Implement autoscaling
- Regular resource optimization
- Monitor unused resources
- Use cost allocation tags

## 11. Troubleshooting

### 11.1. Common Issues
1. Node group scaling issues
2. Pod scheduling problems
3. Network connectivity issues
4. IAM authentication failures
5. Resource quota exceeded

### 11.2. Debugging Commands
```bash
# Check cluster status
eksctl get cluster

# Check node status
kubectl get nodes -o wide

# Check pod status
kubectl get pods --all-namespaces

# Check logs
kubectl logs -f pod-name

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp
```

### 11.3. Health Checks
```bash
# Cluster health
aws eks describe-cluster --name production-eks --region us-west-2

# Node health
kubectl describe node node-name

# Control plane logs
aws eks update-cluster-config --name production-eks --region us-west-2 --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
``` 