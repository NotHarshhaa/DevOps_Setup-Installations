# Install Prometheus and Grafana Monitoring Stack for Kubernetes

This guide will help you set up a complete monitoring stack for your Kubernetes cluster using Prometheus and Grafana via the kube-prometheus-stack Helm chart.

## Prerequisites

- A running Kubernetes cluster
- kubectl configured to communicate with your cluster
- Helm v3 installed
- Admin access to the cluster

## Step 1: Install Helm

Helm is a package manager for Kubernetes. Install it using the official guide: [Helm Installation Guide](https://helm.sh/docs/intro/install/).

Quick installation for Linux:
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify installation:
```bash
helm version
```

## Step 2: Add Prometheus Community Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## Step 3: Create a Namespace for Monitoring

```bash
kubectl create namespace monitoring
```

## Step 4: Install kube-prometheus-stack

The kube-prometheus-stack includes Prometheus, Grafana, Alertmanager, and Kubernetes metrics exporters.

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

Optional: Install with custom values file
```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values custom-values.yaml
```

## Step 5: Verify Installation

```bash
# Check pods in monitoring namespace
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring

# Wait for all pods to be running
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=prometheus -n monitoring --timeout=300s
```

## Step 6: Access Grafana Dashboard

### Port Forwarding

```bash
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

### Retrieve Admin Password

```bash
kubectl get secret --namespace monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### Login

- URL: `http://localhost:3000`
- Username: `admin`
- Password: (use the password retrieved above)

## Step 7: Access Prometheus

```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring
```

Access Prometheus at `http://localhost:9090`

## Step 8: Pre-configured Dashboards

The kube-prometheus-stack comes with pre-configured Grafana dashboards. You can find them under:

1. Log in to Grafana
2. Navigate to Dashboards > Browse
3. Look for dashboards in the "General" folder

Popular pre-installed dashboards include:
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Namespace (Workloads)
- Kubernetes / Compute Resources / Node (Pods)
- Kubernetes / Compute Resources / Pod
- Kubernetes / Compute Resources / Workload

## Step 9: Import Additional Dashboards

If you need additional dashboards, you can import them:

1. In Grafana, go to the "+" menu and select "Import"
2. Enter the dashboard ID or upload a JSON file
3. Configure the data source (Prometheus should be pre-configured)
4. Click "Import"

Recommended dashboard IDs:
- Node Exporter Full: `1860`
- Kubernetes Cluster Monitoring: `12740`
- Kubernetes API Server: `12006`
- Kubernetes Pods/Containers: `6417`
- Kubernetes Deployment Metrics: `10856`

## Step 10: Configure Alerting (Optional)

The kube-prometheus-stack includes Alertmanager. You can customize alert rules and notification channels.

### View Alertmanager UI

```bash
kubectl port-forward svc/prometheus-kube-prometheus-alertmanager 9093:9093 -n monitoring
```

Access at `http://localhost:9093`

### Configure Slack Notifications

Create a custom values file or update the existing installation:

```yaml
# alertmanager-config.yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack-notifications'
      
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - channel: '#monitoring-alerts'
        send_resolved: true
```

Apply the configuration:
```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values alertmanager-config.yaml
```

## Step 11: Uninstall (If Needed)

```bash
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```

## Troubleshooting

**Pods not starting:**
```bash
kubectl describe pod <pod-name> -n monitoring
kubectl logs <pod-name> -n monitoring
```

**Grafana password issues:**
```bash
# Reset admin password
kubectl delete secret prometheus-grafana -n monitoring
# Restart the pod to regenerate the secret
kubectl rollout restart deployment prometheus-grafana -n monitoring
```

**Prometheus not scraping targets:**
```bash
# Check Prometheus targets
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n monitoring
# Visit http://localhost:9090/targets
```

## Additional Resources

- [kube-prometheus-stack Documentation](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Kubernetes Monitoring Best Practices](https://kubernetes.io/docs/tasks/debug/debug-cluster/)

This setup provides a comprehensive monitoring solution for your Kubernetes cluster with Prometheus collecting metrics and Grafana visualizing them with pre-configured dashboards.
