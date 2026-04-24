# AWS Container Services Setup Guide: ECR, ECS, and EKS

![AWS](https://imgur.com/fOUBFH3.png)

Comprehensive guide for setting up Amazon Elastic Container Registry (ECR), Amazon Elastic Container Service (ECS), and Amazon Elastic Kubernetes Service (EKS) with CLI automation and Infrastructure as Code.

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Amazon ECR Setup](#2-amazon-ecr-setup)
3. [Amazon ECS Setup](#3-amazon-ecs-setup)
4. [Amazon EKS Setup](#4-amazon-eks-setup)
5. [Terraform IaC Examples](#5-terraform-iac-examples)
6. [Security Best Practices](#6-security-best-practices)
7. [Monitoring and Logging](#7-monitoring-and-logging)

---

## 1. Prerequisites

### 1.1. Required Tools
```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version

# Install Docker
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

### 1.2. AWS Configuration
```bash
# Configure AWS CLI
aws configure

# Verify configuration
aws sts get-caller-identity
```

---

## 2. Amazon ECR Setup

### 2.1. Create ECR Repository (CLI)

```bash
# Create repository
aws ecr create-repository \
    --repository-name my-app-repo \
    --region us-west-2 \
    --image-scanning-configuration scanOnPush=true \
    --encryption-configuration encryptionType=AES256

# Set lifecycle policy to clean up old images
aws ecr put-lifecycle-policy \
    --repository-name my-app-repo \
    --region us-west-2 \
    --lifecycle-policy-text '{
        "rules": [
            {
                "rulePriority": 1,
                "description": "Keep last 10 images",
                "selection": {
                    "tagStatus": "tagged",
                    "tagPrefixList": ["v"],
                    "countType": "imageCountMoreThan",
                    "countNumber": 10
                },
                "action": {
                    "type": "expire"
                }
            },
            {
                "rulePriority": 2,
                "description": "Delete untagged images older than 7 days",
                "selection": {
                    "tagStatus": "untagged",
                    "countType": "sinceImagePushed",
                    "countUnit": "days",
                    "countNumber": 7
                },
                "action": {
                    "type": "expire"
                }
            }
        ]
    }'
```

### 2.2. Build and Push Docker Image

```bash
# Get ECR login password and authenticate Docker
aws ecr get-login-password --region us-west-2 | \
    docker login --username AWS --password-stdin \
    $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-west-2.amazonaws.com

# Build Docker image
docker build -t my-app:latest .

# Tag image for ECR
docker tag my-app:latest \
    $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-west-2.amazonaws.com/my-app-repo:latest

# Tag with version
docker tag my-app:latest \
    $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-west-2.amazonaws.com/my-app-repo:v1.0.0

# Push images to ECR
docker push $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-west-2.amazonaws.com/my-app-repo:latest
docker push $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-west-2.amazonaws.com/my-app-repo:v1.0.0
```

### 2.3. ECR IAM Policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ECRReadWrite",
            "Effect": "Allow",
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:DescribeRepositories",
                "ecr:GetRepositoryPolicy",
                "ecr:ListImages",
                "ecr:DescribeImages",
                "ecr:BatchGetImage"
            ],
            "Resource": "arn:aws:ecr:us-west-2:ACCOUNT_ID:repository/my-app-repo"
        },
        {
            "Sid": "ECRGetAuthorizationToken",
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## 3. Amazon ECS Setup

### 3.1. Create ECS Cluster (CLI)

```bash
# Create ECS cluster
aws ecs create-cluster \
    --cluster-name my-ecs-cluster \
    --region us-west-2

# Create CloudWatch log group
aws logs create-log-group \
    --log-group-name /ecs/my-app \
    --region us-west-2
```

### 3.2. Create ECS Task Definition

```bash
# Create task definition
aws ecs register-task-definition \
    --family my-app-task \
    --network-mode awsvpc \
    --requires-compatibilities FARGATE \
    --cpu "256" \
    --memory "512" \
    --execution-role-arn ecsTaskExecutionRole \
    --region us-west-2 \
    --container-definitions '[
        {
            "name": "my-app",
            "image": "ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/my-app-repo:latest",
            "cpu": 256,
            "memory": 512,
            "essential": true,
            "portMappings": [
                {
                    "containerPort": 80,
                    "protocol": "tcp"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/my-app",
                    "awslogs-region": "us-west-2",
                    "awslogs-stream-prefix": "ecs"
                }
            },
            "environment": [
                {
                    "name": "ENVIRONMENT",
                    "value": "production"
                }
            ]
        }
    ]'
```

### 3.3. Create ECS Service

```bash
# Create security group for ECS
SG_ID=$(aws ec2 create-security-group \
    --group-name ecs-sg \
    --description "Security group for ECS" \
    --vpc-id vpc-xxxxxxxx \
    --region us-west-2 \
    --query 'GroupId' \
    --output text)

# Allow inbound traffic
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0 \
    --region us-west-2

# Create ECS service
aws ecs create-service \
    --cluster my-ecs-cluster \
    --service-name my-app-service \
    --task-definition my-app-task \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxxxxx,subnet-yyyyyyyy],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" \
    --region us-west-2 \
    --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100"
```

### 3.4. Create Application Load Balancer

```bash
# Create target group
TARGET_ARN=$(aws elbv2 create-target-group \
    --name my-app-tg \
    --protocol HTTP \
    --port 80 \
    --target-type ip \
    --vpc-id vpc-xxxxxxxx \
    --region us-west-2 \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)

# Create load balancer
LB_ARN=$(aws elbv2 create-load-balancer \
    --name my-app-alb \
    --subnets subnet-xxxxxxxx subnet-yyyyyyyy \
    --security-groups $SG_ID \
    --region us-west-2 \
    --query 'LoadBalancers[0].LoadBalancerArn' \
    --output text)

# Create listener
aws elbv2 create-listener \
    --load-balancer-arn $LB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$TARGET_ARN \
    --region us-west-2
```

---

## 4. Amazon EKS Setup

### 4.1. Create EKS Cluster (eksctl)

```yaml
# eks-cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks-cluster
  region: us-west-2
  version: "1.29"

managedNodeGroups:
  - name: managed-nodes
    instanceType: t3.medium
    minSize: 2
    maxSize: 5
    desiredCapacity: 3
    volumeSize: 80
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
eksctl create cluster -f eks-cluster-config.yaml

# Update kubeconfig
aws eks update-kubeconfig --name my-eks-cluster --region us-west-2

# Verify connection
kubectl get nodes
```

### 4.2. Create Kubernetes Deployment for ECR Image

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/my-app-repo:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
```

```bash
# Deploy to EKS
kubectl apply -f deployment.yaml

# Verify deployment
kubectl get pods -l app=my-app
kubectl get services
```

### 4.3. Create Image Pull Secret

```bash
# Create secret for ECR authentication
kubectl create secret docker-registry ecr-secret \
    --docker-server=ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com \
    --docker-username=AWS \
    --docker-password=$(aws ecr get-login-password --region us-west-2) \
    --docker-email=none

# Patch service account to use the secret
kubectl patch serviceaccount default \
    -p '{"imagePullSecrets": [{"name": "ecr-secret"}]}'
```

---

## 5. Terraform IaC Examples

### 5.1. ECR Terraform Module

```hcl
# ecr.tf
resource "aws_ecr_repository" "my_app" {
  name                 = "my-app-repo"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }
}

resource "aws_ecr_lifecycle_policy" "my_app" {
  repository = aws_ecr_repository.my_app.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 10
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

### 5.2. EKS Terraform Module

```hcl
# eks.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "my-eks-cluster"
  cluster_version = "1.29"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  eks_managed_node_groups = {
    main = {
      name           = "managed-nodes"
      instance_types = ["t3.medium"]
      min_size       = 2
      max_size       = 5
      desired_size   = 3

      labels = {
        role = "worker"
      }

      tags = {
        Environment = "production"
      }
    }
  }

  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
  }
}
```

---

## 6. Security Best Practices

### 6.1. IAM Roles and Policies

```bash
# Create IAM role for ECS task execution
aws iam create-role \
    --role-name ecsTaskExecutionRole \
    --assume-role-policy-document file://trust-policy.json

# Attach managed policy
aws iam attach-role-policy \
    --role-name ecsTaskExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### 6.2. Security Groups

```bash
# Restrict security group to specific CIDR
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr 10.0.0.0/16 \
    --region us-west-2
```

### 6.3. Secrets Management

```bash
# Store sensitive data in AWS Secrets Manager
aws secretsmanager create-secret \
    --name my-app/database \
    --secret-string '{"username":"admin","password":"securepassword"}' \
    --region us-west-2

# Retrieve secret in application
aws secretsmanager get-secret-value \
    --secret-id my-app/database \
    --region us-west-2
```

---

## 7. Monitoring and Logging

### 7.1. CloudWatch Logs

```bash
# Enable CloudWatch logging for ECS
aws ecs put-cluster-configuration \
    --cluster my-ecs-cluster \
    --execute-configuration '{"executeCommandConfiguration":{"logging":"OVERRIDE","logConfiguration":{"cloudWatchLogGroupName":"/ecs/my-app"}}}' \
    --region us-west-2
```

### 7.2. CloudWatch Metrics

```bash
# Create CloudWatch dashboard
aws cloudwatch put-dashboard \
    --dashboard-name my-app-dashboard \
    --dashboard-body file://dashboard.json \
    --region us-west-2
```

### 7.3. X-Ray Tracing

```bash
# Enable AWS X-Ray for ECS
aws ecs put-account-setting \
    --name accountSettingName:containerInsights \
    --value enabled \
    --region us-west-2
```

---

## 8. CI/CD Integration

### 8.1. GitHub Actions Example

```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build and push Docker image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: my-app-repo
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

    - name: Deploy to EKS
      run: |
        aws eks update-kubeconfig --name my-eks-cluster --region us-west-2
        kubectl set image deployment/my-app my-app=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

---

## Troubleshooting

### Common Issues

1. **ECR Pull Errors**
   ```bash
   # Re-authenticate Docker
   aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com
   ```

2. **EKS Node Not Ready**
   ```bash
   # Check node status
   kubectl describe node <node-name>
   
   # Check node logs
   kubectl logs -n kube-system -l k8s-app=aws-node
   ```

3. **ECS Task Stuck in PENDING**
   ```bash
   # Check task events
   aws ecs describe-tasks --cluster my-ecs-cluster --tasks <task-id> --region us-west-2
   ```

---

## References

- [AWS ECR Documentation](https://docs.aws.amazon.com/AmazonECR/latest/userguide/)
- [AWS ECS Documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/)
- [eksctl Documentation](https://eksctl.io/)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
