# Kubernetes Monitoring Stack Setup Guide

![banner](https://miro.medium.com/v2/resize:fit:2000/1*5rcKvi6ZSwevNQxroWGtlA.png)

This comprehensive guide covers setting up a production-grade monitoring stack for Kubernetes using Prometheus, Grafana, and related components.

## 1. Prerequisites

### 1.1. Required Tools
- Kubernetes cluster (v1.24+ recommended)
- Helm v3.8+
- kubectl configured for your cluster
- Cluster admin privileges

### 1.2. Resource Requirements
- Minimum 4 CPU cores available
- Minimum 8GB RAM available
- At least 50GB storage for monitoring data (adjust based on retention needs)
- Network connectivity for pulling container images

## 2. Installing Prometheus Stack using Helm

The kube-prometheus-stack is a collection of Kubernetes manifests, Grafana dashboards, and Prometheus rules combined with documentation and scripts to provide easy-to-operate end-to-end Kubernetes cluster monitoring.

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
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set prometheus.prometheusSpec.resources.requests.memory=2Gi \
  --set prometheus.prometheusSpec.resources.requests.cpu=1000m \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.size=10Gi
```

### 2.4. Install with Custom Values File

For production environments, use a values file for better configuration management:

```yaml
# prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    retentionSize: 50GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    resources:
      requests:
        memory: 4Gi
        cpu: 2000m
      limits:
        memory: 8Gi
        cpu: 4000m

grafana:
  persistence:
    enabled: true
    size: 20Gi
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: 500m

alertmanager:
  enabled: true
  persistence:
    enabled: true
    size: 5Gi
```

Install with custom values:
```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml
```

### 2.5. Verify Installation

```bash
# Check all pods are running
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring

# Check persistent volume claims
kubectl get pvc -n monitoring

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=prometheus -n monitoring --timeout=300s
```

## 3. Configuring Grafana

### 3.1. Access Grafana Dashboard

```bash
# Port forward Grafana service
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

### 3.2. Retrieve Admin Password

```bash
kubectl get secret --namespace monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Default credentials:
- Username: admin
- Password: (retrieve using command above)

### 3.3. Pre-installed Dashboards

The kube-prometheus-stack includes pre-configured dashboards:
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace (Workloads)
- Kubernetes / Compute Resources / Node (Pods)
- Kubernetes / Compute Resources / Pod
- Kubernetes / Compute Resources / Workload
- Kubernetes / Networking / Cluster
- Kubernetes / Networking / Namespace (Workloads)
- Kubernetes / Persistent Volumes
- Kubernetes / Pods
- Kubernetes / StatefulSet
- Prometheus / Prometheus
- Node Exporter / Nodes
- Node Exporter / Node Exporter Full

### 3.4. Import Additional Dashboards

Recommended dashboard IDs to import:
1. Node Exporter Full (ID: 1860)
2. Kubernetes Cluster Monitoring (ID: 12740)
3. Kubernetes API Server (ID: 12006)
4. Kubernetes Pods/Containers (ID: 6417)
5. Kubernetes Deployment Metrics (ID: 10856)
6. Kubernetes Cluster Health (ID: 7249)

## 4. Configuring Alertmanager

### 4.1. Access Alertmanager UI

```bash
kubectl port-forward svc/prometheus-kube-prometheus-alertmanager 9093:9093 -n monitoring
```

### 4.2. Configure Slack Notifications

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
      group_by: ['alertname', 'namespace', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack-notifications'
      routes:
        - match:
            severity: critical
          receiver: 'slack-critical'
          continue: true

    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#monitoring-alerts'
        send_resolved: true
        title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        title_link: 'https://grafana.example.com/d/prometheus'
        text: "{{ range .Alerts }}*Alert:* {{ .Annotations.summary }}\n*Description:* {{ .Annotations.description }}\n*Severity:* {{ .Labels.severity }}\n*Namespace:* {{ .Labels.namespace }}\n{{ end }}"

    - name: 'slack-critical'
      slack_configs:
      - channel: '#critical-alerts'
        send_resolved: true
        title: '🚨 CRITICAL: {{ .CommonLabels.alertname }}'
        text: "{{ range .Alerts }}*Alert:* {{ .Annotations.summary }}\n*Description:* {{ .Annotations.description }}\n*Severity:* {{ .Labels.severity }}\n*Namespace:* {{ .Labels.namespace }}\n*Pod:* {{ .Labels.pod }}\n{{ end }}"
```

Apply the configuration:
```bash
kubectl apply -f alertmanager-config.yaml
```

### 4.3. Configure Email Notifications

```yaml
    receivers:
    - name: 'email-notifications'
      email_configs:
      - to: 'alerts@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'password'
        headers:
          Subject: '[Alert] {{ .GroupLabels.alertname }}'
```

## 5. Setting up Node Exporter

**Note:** The kube-prometheus-stack already includes Node Exporter. The following is for custom deployment if needed.

### 5.1. Deploy Node Exporter DaemonSet

```yaml
# node-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
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
        image: prom/node-exporter:v1.8.0
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($|/)
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true
          mountPropagation: HostToContainer
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
      tolerations:
      - effect: NoSchedule
        operator: Exists
```

### 5.2. Create Node Exporter Service

```yaml
# node-exporter-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
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

## 6. Custom Metrics Collection

### 6.1. Configure Custom Metrics API with Prometheus Adapter

```bash
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local \
  --set prometheus.port=9090
```

### 6.2. Add Custom Scrape Configs

```yaml
# custom-scrape-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-additional-scrape-config
  namespace: monitoring
data:
  custom-scrape.yaml: |
    - job_name: 'custom-application'
      scrape_interval: 30s
      scrape_timeout: 10s
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - application
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
```

Apply and update Prometheus:
```bash
kubectl apply -f custom-scrape-config.yaml
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.additionalScrapeConfigs[0]=secret:prometheus-additional-scrape-config
```

## 7. Best Practices

### 7.1. Resource Management
- Set appropriate resource requests and limits for all components
- Configure retention periods based on storage capacity (15-30 days typical)
- Use persistent storage with appropriate storage class
- Implement regular backup strategies for Grafana dashboards and configurations
- Monitor resource usage and scale accordingly

### 7.2. Security
- Enable RBAC with least privilege principles
- Use TLS for all communications where possible
- Implement network policies to restrict traffic
- Keep all components updated with security patches
- Secure sensitive information in Kubernetes secrets
- Use service accounts with minimal permissions
- Enable authentication for Grafana (LDAP, OAuth, etc.)

### 7.3. Alert Management
- Set meaningful alert thresholds based on baselines
- Configure proper alert routing and escalation
- Implement alert aggregation to reduce noise
- Document alert response procedures and runbooks
- Regular alert review and maintenance (quarterly recommended)
- Use alert silencing for planned maintenance
- Implement on-call rotation and escalation policies

### 7.4. Performance Optimization
- Optimize scrape intervals (15-60s typical, avoid <10s)
- Use recording rules for complex and frequently used queries
- Implement proper data retention policies
- Use appropriate instance sizing based on workload
- Regular performance monitoring and tuning
- Consider using Thanos for long-term storage and federation
- Implement remote write for multi-cluster setups

### 7.5. High Availability
- Deploy Prometheus with replica support
- Use persistent storage with redundancy
- Implement multi-zone deployment for critical workloads
- Configure alertmanager with clustering
- Use load balancers for Grafana access

## 8. Troubleshooting

### 8.1. Common Issues
1. High memory usage in Prometheus
2. Scrape failures or targets down
3. Alertmanager configuration issues
4. Grafana dashboard loading problems
5. Storage persistence issues
6. Time synchronization problems
7. Slow query performance

### 8.2. Debugging Commands

```bash
# Check Prometheus logs
kubectl logs -f prometheus-prometheus-kube-prometheus-prometheus-0 -n monitoring

# Check Alertmanager logs
kubectl logs -f alertmanager-prometheus-kube-prometheus-alertmanager-0 -n monitoring

# Check Grafana logs
kubectl logs -f deployment/prometheus-grafana -n monitoring

# Check all monitoring pods
kubectl get pods -n monitoring -o wide

# Check persistent volume claims
kubectl get pvc -n monitoring

# Check events in monitoring namespace
kubectl get events -n monitoring --sort-by='.lastTimestamp'

# Port forward to Prometheus UI
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring

# Port forward to Alertmanager UI
kubectl port-forward svc/prometheus-kube-prometheus-alertmanager 9093:9093 -n monitoring
```

### 8.3. Health Checks

```bash
# Check Prometheus targets
curl localhost:9090/api/v1/targets

# Check Prometheus configuration
curl localhost:9090/api/v1/status/config

# Check Alertmanager status
curl localhost:9093/api/v1/status

# Check Node Exporter metrics
curl localhost:9100/metrics

# Check Prometheus TSDB stats
curl localhost:9090/api/v1/status/tsdb
```

### 8.4. Common Fixes

**Prometheus OOM (Out of Memory)**
```bash
# Increase memory limits
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.resources.limits.memory=8Gi
```

**Grafana login issues**
```bash
# Reset admin password
kubectl delete secret prometheus-grafana -n monitoring
kubectl rollout restart deployment/prometheus-grafana -n monitoring
```

**Storage full**
```bash
# Reduce retention period
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=7d
```

## 9. Uninstalling

```bash
# Remove the Helm release
helm uninstall prometheus -n monitoring

# Remove the namespace
kubectl delete namespace monitoring

# Remove persistent volumes (if desired)
kubectl delete pvc -n monitoring --all
```

## 10. Additional Resources

- [kube-prometheus-stack Documentation](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Kubernetes Monitoring Best Practices](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [Thanos Documentation](https://thanos.io/)

This setup provides a comprehensive monitoring solution for your Kubernetes cluster with Prometheus collecting metrics, Grafana visualizing them, and Alertmanager handling notifications.