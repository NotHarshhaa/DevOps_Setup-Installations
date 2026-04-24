# Setting up a 10-Microservices Application Deployment on Amazon EKS

![EKS Microservices](https://d1.awsstatic.com/hero-images/Container-EKS_Hero-Image_4x.5c8665c605d5223906d6b3d6e8d1d8a5d8b5a5c8.png)

Comprehensive guide for deploying a 10-microservices application architecture on Amazon EKS with modern DevOps practices, observability, and GitOps.

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [EKS Cluster Setup](#2-eks-cluster-setup)
3. [Infrastructure Components](#3-infrastructure-components)
4. [Microservices Architecture](#4-microservices-architecture)
5. [Deployment Strategies](#5-deployment-strategies)
6. [Observability Stack](#6-observability-stack)
7. [Service Mesh](#7-service-mesh)
8. [CI/CD Pipeline](#8-cicd-pipeline)
9. [GitOps with ArgoCD](#9-gitops-with-argocd)
10. [Security Configuration](#10-security-configuration)
11. [Scaling and HA](#11-scaling-and-ha)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Prerequisites

### 1.1. Required Tools
```bash
# AWS CLI v2.15+
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# kubectl v1.29+
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# eksctl v0.169+
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Helm v3.14+
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Docker/Podman
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
```

### 1.2. AWS Configuration
```bash
aws configure
aws sts get-caller-identity
```

---

## 2. EKS Cluster Setup

### 2.1. Cluster Configuration

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: microservices-cluster
  region: us-west-2
  version: "1.29"

managedNodeGroups:
  - name: primary-nodes
    instanceType: t3.large
    minSize: 3
    maxSize: 10
    desiredCapacity: 3
    volumeSize: 100
    labels:
      node-type: primary
    iam:
      withAddonPolicies:
        autoScaler: true
        albIngress: true
        cloudWatch: true
        externalDNS: true

  - name: spot-nodes
    instanceTypes: ["t3a.large", "t3.large", "t3.xlarge"]
    spot: true
    minSize: 2
    maxSize: 5
    desiredCapacity: 2
    labels:
      node-type: spot
      workload: batch

vpc:
  cidr: "10.0.0.0/16"
  nat:
    gateway: HighlyAvailable
  clusterEndpoints:
    publicAccess: true
    privateAccess: true

cloudWatch:
  clusterLogging:
    enableTypes: ["*"]
```

```bash
# Create cluster
eksctl create cluster -f cluster-config.yaml

# Update kubeconfig
aws eks update-kubeconfig --name microservices-cluster --region us-west-2

# Verify
kubectl get nodes
```

---

## 3. Infrastructure Components

### 3.1. AWS Load Balancer Controller

```bash
# Add Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Create IAM policy
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json

# Create IAM role and service account
eksctl create iamserviceaccount \
  --cluster=microservices-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --override-existing-serviceaccounts

# Install controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=microservices-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.7.1
```

### 3.2. External DNS

```bash
# Create IAM policy
cat <<EOF > external-dns-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF

aws iam create-policy \
    --policy-name ExternalDNSPolicy \
    --policy-document file://external-dns-policy.json

# Create service account
eksctl create iamserviceaccount \
  --cluster=microservices-cluster \
  --namespace=default \
  --name=external-dns \
  --attach-policy-arn=arn:aws:iam::ACCOUNT_ID:policy/ExternalDNSPolicy \
  --approve \
  --override-existing-serviceaccounts

# Install External DNS
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install external-dns bitnami/external-dns \
  --set provider=aws \
  --set aws.region=us-west-2 \
  --set txtOwnerId=microservices-cluster \
  --set domainFilters={example.com}
```

### 3.3. Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl get apiservice v1beta1.metrics.k8s.io
```

### 3.4. Cluster Autoscaler

```bash
# Install via Helm
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set cloudProvider=aws \
  --set autoDiscovery.clusterName=microservices-cluster \
  --set awsRegion=us-west-2 \
  --set rbac.create=true \
  --set rbac.serviceAccount.create=true
```

---

## 4. Microservices Architecture

### 4.1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                       API Gateway (ALB)                     │
└─────────────────────┬───────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
┌───────▼──────┐ ┌───▼────┐ ┌─────▼──────┐
│  Frontend    │ │ Auth   │ │  Product   │
│  Service     │ │ Service│ │  Service   │
└───────┬──────┘ └───┬────┘ └─────┬──────┘
        │            │            │
        └────────────┼────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
┌───────▼──────┐ ┌──▼───┐ ┌──────▼──────┐
│   Order      │ │ Cart │ │  Payment    │
│   Service    │ │ Svc  │ │  Service    │
└───────┬──────┘ └──┬───┘ └──────┬──────┘
        │           │            │
        └───────────┼────────────┘
                    │
        ┌───────────┼────────────┐
        │           │            │
┌───────▼──────┐ ┌─▼────┐ ┌─────▼──────┐
│ Notification │ │ User │ │ Inventory  │
│   Service    │ │ Svc  │ │  Service   │
└───────┬──────┘ └──┬───┘ └─────┬──────┘
        │           │            │
        └───────────┼────────────┘
                    │
        ┌───────────▼────────────┐
        │     Analytics Svc      │
        └────────────────────────┘
```

### 4.2. Sample Microservice Deployment

```yaml
# api-gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: microservices
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
        version: v1
    spec:
      containers:
      - name: api-gateway
        image: ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/api-gateway:v1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: AUTH_SERVICE_URL
          value: "http://auth-service:8080"
        - name: PRODUCT_SERVICE_URL
          value: "http://product-service:8080"
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: microservices
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 4.3. Create Namespace

```bash
# Create namespace for microservices
kubectl create namespace microservices

# Create image pull secret
kubectl create secret docker-registry ecr-secret \
    --docker-server=ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com \
    --docker-username=AWS \
    --docker-password=$(aws ecr get-login-password --region us-west-2) \
    --docker-email=none \
    -n microservices

# Patch default service account
kubectl patch serviceaccount default \
    -n microservices \
    -p '{"imagePullSecrets": [{"name": "ecr-secret"}]}'
```

---

## 5. Deployment Strategies

### 5.1. Rolling Update Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: microservices
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/auth-service:v1.0.0
        # ... rest of configuration
```

### 5.2. Blue-Green Deployment with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: product-service
  namespace: microservices
spec:
  replicas: 5
  strategy:
    blueGreen:
      activeService: product-service-active
      previewService: product-service-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: product-service-preview
      postPromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: product-service-active
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
      - name: product-service
        image: ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/product-service:v1.0.0
        ports:
        - containerPort: 8080
```

### 5.3. Canary Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
  namespace: microservices
spec:
  replicas: 5
  strategy:
    canary:
      canaryService: payment-service-canary
      stableService: payment-service-stable
      trafficAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: payment-service-canary
      steps:
      - setWeight: 10
      - pause: {duration: 10m}
      - setWeight: 25
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/payment-service:v1.0.0
        ports:
        - containerPort: 8080
```

---

## 6. Observability Stack

### 6.1. Prometheus and Grafana

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace \
    --set grafana.adminPassword=admin \
    --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
    --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false

# Install Grafana dashboards
kubectl create configmap grafana-dashboards \
    --from-file=./dashboards/ \
    -n monitoring

# Port forward to access Grafana
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

### 6.2. Service Monitors

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-gateway
  namespace: microservices
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: api-gateway
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

### 6.3. Loki for Logs

```bash
# Install Loki
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
    --namespace monitoring \
    --set loki.persistence.enabled=true \
    --set loki.persistence.size=20Gi \
    --set promtail.enabled=true
```

### 6.4. Tempo for Tracing

```bash
# Install Tempo
helm install tempo grafana/tempo \
    --namespace monitoring \
    --set tempo.multitenancyEnabled=false \
    --set receiver.jaeger.enabled=true \
    --set receiver.otlp.enabled=true
```

---

## 7. Service Mesh

### 7.1. Install Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*/
export PATH=$PWD/bin:$PATH

# Install Istio
istioctl install --set profile=default -y

# Enable automatic sidecar injection
kubectl label namespace microservices istio-injection=enabled
```

### 7.2. VirtualService Example

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-gateway
  namespace: microservices
spec:
  hosts:
  - api-gateway
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: api-gateway
        subset: v2
  - route:
    - destination:
        host: api-gateway
        subset: v1
      weight: 100
```

### 7.3. DestinationRule

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-gateway
  namespace: microservices
spec:
  host: api-gateway
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

---

## 8. CI/CD Pipeline

### 8.1. GitHub Actions Workflow

```yaml
# .github/workflows/deploy-microservices.yml
name: Deploy Microservices

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api-gateway, auth-service, product-service, order-service, cart-service, payment-service, notification-service, user-service, inventory-service, analytics-service]
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
        ECR_REPOSITORY: ${{ matrix.service }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG ./services/${{ matrix.service }}
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
    
    - name: Update Kubernetes deployment
      run: |
        aws eks update-kubeconfig --name microservices-cluster --region us-west-2
        kubectl set image deployment/${{ matrix.service }} ${{ matrix.service }}=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG -n microservices
```

---

## 9. GitOps with ArgoCD

### 9.1. Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial password
argocd admin initial-password -n argocd
```

### 9.2. Application Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: microservices-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/microservices-k8s
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: microservices
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## 10. Security Configuration

### 10.1. Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

### 10.2. Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: microservices
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-gateway
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: api-gateway
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: auth-service
    ports:
    - protocol: TCP
      port: 8080
```

### 10.3. Secrets Management

```bash
# Install AWS Secrets Manager CSI Driver
helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
helm repo update

helm install secrets-store-csi-driver secrets-store-csi-driver/secrets-store-csi-driver \
    --namespace kube-system

kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml

# Create SecretProviderClass
cat <<EOF > secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
  namespace: microservices
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "microservices/database"
        objectType: "secretsmanager"
        jmesPath:
          - path: "username"
            objectAlias: "db_username"
          - path: "password"
            objectAlias: "db_password"
  secretObjects:
  - secretName: database-credentials
    type: Opaque
    data:
    - objectName: db_username
      key: username
    - objectName: db_password
      key: password
EOF

kubectl apply -f secret-provider-class.yaml
```

---

## 11. Scaling and HA

### 11.1. Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: auth-service-hpa
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 2
        periodSeconds: 30
      selectPolicy: Max
```

### 11.2. Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: auth-service-pdb
  namespace: microservices
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: auth-service
```

### 11.3. Multi-AZ Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: microservices
spec:
  replicas: 6
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: order-service
      containers:
      - name: order-service
        image: ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/order-service:v1.0.0
        ports:
        - containerPort: 8080
```

---

## 12. Troubleshooting

### 12.1. Common Issues

**Pod Not Ready**
```bash
kubectl describe pod <pod-name> -n microservices
kubectl logs <pod-name> -n microservices
```

**Service Not Accessible**
```bash
kubectl get endpoints <service-name> -n microservices
kubectl describe svc <service-name> -n microservices
```

**Helm Release Issues**
```bash
helm list -n microservices
helm status <release-name> -n microservices
helm rollback <release-name> -n microservices
```

**Network Policy Issues**
```bash
kubectl get networkpolicies -n microservices
kubectl describe networkpolicy <policy-name> -n microservices
```

### 12.2. Debugging Commands

```bash
# Check cluster health
kubectl cluster-info
kubectl get componentstatuses

# Check node status
kubectl get nodes -o wide
kubectl describe node <node-name>

# Check pod status
kubectl get pods -n microservices -o wide
kubectl get pods -n microservices --sort-by=.metadata.creationTimestamp

# Check events
kubectl get events -n microservices --sort-by=.metadata.creationTimestamp

# Port forward for debugging
kubectl port-forward svc/<service-name> 8080:80 -n microservices

# Exec into pod
kubectl exec -it <pod-name> -n microservices -- /bin/bash

# Check resource usage
kubectl top nodes
kubectl top pods -n microservices
```

### 12.3. Logs Aggregation

```bash
# View logs for all pods in a deployment
kubectl logs -l app=auth-service -n microservices --tail=100

# View logs from multiple containers
kubectl logs <pod-name> -n microservices -c <container-name>

# Stream logs
kubectl logs -f <pod-name> -n microservices

# View previous logs if pod crashed
kubectl logs <pod-name> -n microservices --previous
```

---

## Best Practices Summary

1. **Use Namespaces**: Organize microservices into separate namespaces for better isolation and management
2. **Resource Limits**: Always set CPU and memory requests/limits for all pods
3. **Health Checks**: Implement liveness and readiness probes for all services
4. **Auto-scaling**: Configure HPA for all critical services
5. **Monitoring**: Implement comprehensive monitoring with Prometheus and Grafana
6. **Logging**: Centralize logs with Loki or CloudWatch
7. **Tracing**: Use distributed tracing with Tempo or Jaeger
8. **Security**: Implement network policies, pod security standards, and secrets management
9. **GitOps**: Use ArgoCD or Flux for declarative deployments
10. **Testing**: Implement comprehensive testing in CI/CD pipeline
11. **Documentation**: Maintain up-to-date documentation for all services
12. **Disaster Recovery**: Implement backup and recovery strategies with Velero

---

## References

- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Istio Documentation](https://istio.io/latest/docs/)
- [ArgoCD Documentation](https://argoproj.github.io/argo-cd/)
- [Prometheus Documentation](https://prometheus.io/docs/)
