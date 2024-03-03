# Install Helm and set up Prometheus and Grafana for Kubernetes cluster

#### To install Helm and set up Prometheus and Grafana for a Kubernetes cluster, follow these steps:

### 1. Install Helm:

Helm is a package manager for Kubernetes. You can install Helm by following the instructions from the official Helm documentation: [Helm Installation Guide](https://helm.sh/docs/intro/install/).

### 2. Install Prometheus with Helm:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack
```

### 3. Install Grafana with Helm:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana
```

### 4. Accessing Grafana Dashboard:

Retrieve the admin password:

```bash
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Port-forward the Grafana service:

```bash
kubectl port-forward --namespace default svc/grafana 3000:80
```

Now, you can access Grafana at `http://localhost:3000` using the username `admin` and the password obtained in the previous step.

### 5. Configure Prometheus as a Data Source in Grafana:

- Log in to Grafana.
- Navigate to Configuration > Data Sources.
- Click on "Add data source".
- Choose Prometheus.
- Configure the URL to point to your Prometheus server (usually `http://prometheus-server:80` if you're using default settings).
- Click "Save & Test".

### 6. Import Kubernetes Dashboards to Grafana:

Grafana provides pre-built dashboards for Kubernetes monitoring. You can import these dashboards to visualize your Kubernetes cluster metrics. Here's one way to do it:

- In Grafana, go to the "+" menu and select "Import".
- Use the following dashboard IDs to import Kubernetes dashboards:
  - Kubernetes Cluster Monitoring: `1621`
  - Kubernetes Deployment Metrics: `10856`
  - Kubernetes Node Metrics: `1860`
  - Kubernetes Pod Metrics: `6417`
- Customize the dashboards as needed.

### 7. Explore and Monitor:

You should now have Prometheus collecting metrics from your Kubernetes cluster and Grafana visualizing these metrics. Explore the dashboards and customize them according to your monitoring needs.

This setup provides basic monitoring capabilities. Depending on your requirements, you may need to further customize Prometheus and Grafana configurations and add additional exporters to collect specific metrics.
