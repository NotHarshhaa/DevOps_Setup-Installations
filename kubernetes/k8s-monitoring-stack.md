# Kubernetes Monitoring Stack Setup Guide

![banner](https://miro.medium.com/v2/resize:fit:2000/1*5rcKvi6ZSwevNQxroWGtlA.png)

## 1. Prerequisites

### 1.1. Required Tools
- Kubernetes cluster (v1.19+)
- Helm v3
- kubectl
- Access to cluster with admin privileges

### 1.2. Resource Requirements
- Minimum 4 CPU cores available
- Minimum 8GB RAM available
- At least 50GB storage for monitoring data

## 2. Installing Prometheus Stack using Helm

### 2.1. Add Prometheus Community Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2.2. Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

### 2.3. Install Prometheus Stack

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.enabled=true \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

### 2.4. Verify Installation

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

## 3. Configuring Grafana

### 3.1. Access Grafana Dashboard

```bash
# Port forward Grafana service
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

Default credentials:
- Username: admin
- Password: prom-operator

### 3.2. Essential Dashboards to Import
1. Node Exporter Full (ID: 1860)
2. Kubernetes Cluster Monitoring (ID: 12740)
3. Kubernetes API Server (ID: 12006)
4. Kubernetes Pods/Containers (ID: 6417)

### 3.3. Configure Alert Notifications

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-prometheus-kube-prometheus-alertmanager
  namespace: monitoring
stringData:
  alertmanager.yaml: |-
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    
    route:
      group_by: ['namespace', 'alertname']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack-notifications'
      
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#monitoring-alerts'
        send_resolved: true
        title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: "{{ range .Alerts }}*Alert:* {{ .Annotations.summary }}\n*Description:* {{ .Annotations.description }}\n*Severity:* {{ .Labels.severity }}\n{{ end }}"
```

Apply the configuration:
```bash
kubectl apply -f alertmanager-config.yaml
```

## 4. Setting up Node Exporter

### 4.1. Deploy Node Exporter DaemonSet

```yaml
# node-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.3.1
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

### 4.2. Create Node Exporter Service

```yaml
# node-exporter-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9100"
spec:
  selector:
    app: node-exporter
  ports:
  - name: metrics
    port: 9100
    targetPort: 9100
```

Apply configurations:
```bash
kubectl apply -f node-exporter.yaml
kubectl apply -f node-exporter-service.yaml
```

## 5. Custom Metrics Collection

### 5.1. Configure Custom Metrics API

```yaml
# custom-metrics.yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.custom.metrics.k8s.io
spec:
  service:
    name: prometheus-adapter
    namespace: monitoring
  group: custom.metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

### 5.2. Add Custom Scrape Configs

```yaml
# custom-scrape-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-additional-scrape-config
  namespace: monitoring
data:
  custom-scrape.yaml: |
    - job_name: 'custom-endpoints'
      scrape_interval: 30s
      static_configs:
        - targets: ['custom-service:8080']
```

## 6. Best Practices

### 6.1. Resource Management
- Set appropriate resource requests and limits
- Configure retention periods based on storage capacity
- Use persistent storage for Prometheus data
- Implement proper backup strategies

### 6.2. Security
- Enable RBAC
- Use secure communication (TLS)
- Implement network policies
- Regular security updates
- Secure sensitive information in secrets

### 6.3. Alert Management
- Set meaningful alert thresholds
- Configure proper alert routing
- Implement alert aggregation
- Document alert response procedures
- Regular alert review and maintenance

### 6.4. Performance Optimization
- Optimize scrape intervals
- Use recording rules for complex queries
- Implement proper data retention policies
- Use appropriate instance sizing
- Regular performance monitoring

## 7. Troubleshooting

### 7.1. Common Issues
1. High memory usage in Prometheus
2. Scrape failures
3. Alert manager configuration issues
4. Grafana dashboard loading problems
5. Storage persistence issues

### 7.2. Debugging Commands
```bash
# Check Prometheus logs
kubectl logs -f prometheus-prometheus-kube-prometheus-prometheus-0 -n monitoring

# Check Alertmanager logs
kubectl logs -f alertmanager-prometheus-kube-prometheus-alertmanager-0 -n monitoring

# Check Grafana logs
kubectl logs -f $(kubectl get pods -n monitoring | grep grafana | awk '{print $1}') -n monitoring

# Check metrics endpoints
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring
```

### 7.3. Health Checks
```bash
# Check Prometheus targets
curl localhost:9090/api/v1/targets

# Check Alertmanager status
curl localhost:9093/api/v1/status

# Check Node Exporter metrics
curl localhost:9100/metrics
``` 