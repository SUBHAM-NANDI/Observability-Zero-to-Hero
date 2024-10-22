#### 1. **Prerequisites**

Please ensure you have the following prerequisites:

- A Kubernetes cluster running (EKS, GKE, Minikube, etc.)
- `kubectl` CLI installed and configured to communicate with the Kubernetes cluster
- Helm installed on your machine for deploying the Prometheus stack

---

#### 2. **Install Prometheus Stack Using Helm**

We will use Helm to install the Prometheus stack, which includes Prometheus, Grafana, Alertmanager, and other components like `kube-state-metrics` and `node_exporter`.

1. **Add the Helm Repository:**
   
   First, add the `prometheus-community` Helm chart repository.

   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. **Install the Prometheus Stack:**

   Now, install the Prometheus stack. This command will install Prometheus, Grafana, Alertmanager, and other related services in a `monitoring` namespace.

   ```bash
   kubectl create namespace monitoring
   helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring
   ```

   This installs all components, including:
   - **Prometheus** for scraping and storing metrics
   - **Grafana** for visualizing metrics
   - **Alertmanager** for managing alerts
   - **kube-state-metrics** for Kubernetes metrics
   - **node_exporter** for node-level metrics

3. **Verify the Installation:**

   Run the following command to ensure all pods are up and running:

   ```bash
   kubectl get pods -n monitoring
   ```

   You should see pods for Prometheus, Grafana, Alertmanager, `node_exporter`, and `kube-state-metrics`.

---

#### 3. **Expose Services Using `kubectl port-forward`**

Since your Kubernetes cluster might not have a LoadBalancer or Ingress configured, we will use `kubectl port-forward` to access the Prometheus, Grafana, and Alertmanager web interfaces.

##### Expose Prometheus:

```bash
kubectl port-forward svc/prometheus-stack-kube-prometheus-prometheus -n monitoring 9090:9090
```

- This command forwards local port `9090` to Prometheus running in Kubernetes.
- You can access Prometheus by opening `http://localhost:9090` in your browser.

##### Expose Grafana:

```bash
kubectl port-forward svc/prometheus-stack-grafana -n monitoring 3000:80
```

- This command forwards local port `3000` to Grafana running in Kubernetes.
- You can access Grafana by opening `http://localhost:3000` in your browser.

   - **Grafana Credentials:** The default username is `admin` and the password can be obtained by running:

     ```bash
     kubectl get secret --namespace monitoring prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
     ```

##### Expose Alertmanager:

```bash
kubectl port-forward svc/prometheus-stack-kube-prometheus-alertmanager -n monitoring 9093:9093
```

- This command forwards local port `9093` to Alertmanager running in Kubernetes.
- You can access Alertmanager by opening `http://localhost:9093` in your browser.

---

#### 4. **Understanding Exporters and Metrics Collection**

Prometheus collects metrics from Kubernetes using **exporters**. Two critical exporters included in the Prometheus stack are:

- **node_exporter:** Provides hardware and OS-level metrics, such as CPU, memory, and disk usage for each Kubernetes node.
- **kube-state-metrics:** Collects Kubernetes object metrics, like pod status, node status, container restarts, etc.

##### Checking `node_exporter` and `kube-state-metrics` Pods:

To verify that these exporters are running:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter
kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics
```

These should display the respective pods.

##### Accessing `node_exporter` Metrics:

`node_exporter` runs as a **DaemonSet** (i.e., one pod per node). It exposes metrics about each node in the cluster.

To check metrics for `node_exporter`:

1. First, get the service IP of `node_exporter`:

   ```bash
   kubectl get svc -n monitoring
   ```

   Look for `prometheus-stack-kube-prometheus-node-exporter`, which will have a **ClusterIP**.

2. To access `node_exporter` metrics:

   ```bash
   kubectl port-forward svc/prometheus-stack-kube-prometheus-node-exporter -n monitoring 9100:9100
   ```

3. Then, open `http://localhost:9100/metrics` to view the raw metrics exposed by `node_exporter`.

##### Accessing `kube-state-metrics`:

`kube-state-metrics` provides information about the state of Kubernetes objects (pods, services, deployments, etc.).

To access `kube-state-metrics`:

1. Get the service IP for `kube-state-metrics`:

   ```bash
   kubectl get svc -n monitoring
   ```

   Look for `prometheus-stack-kube-state-metrics`, which will have a **ClusterIP**.

2. To access `kube-state-metrics` metrics:

   ```bash
   kubectl port-forward svc/prometheus-stack-kube-state-metrics -n monitoring 8080:8080
   ```

3. Then, open `http://localhost:8080/metrics` to view the raw metrics exposed by `kube-state-metrics`.

---

#### 5. **Querying Metrics in Prometheus**

Prometheus provides a powerful query language called **PromQL** to extract and analyze metrics. Let's see some examples:

- **Get total container restarts in the cluster:**

  ```bash
  kube_pod_container_status_restarts_total
  ```

  This query returns how many times containers have restarted in all namespaces.

- **Filter restarts for a specific namespace (e.g., `default`):**

  ```bash
  kube_pod_container_status_restarts_total{namespace="default"}
  ```

- **Get CPU usage for all nodes:**

  ```bash
  node_cpu_seconds_total
  ```

---

#### 6. **Create Grafana Dashboards**

Grafana helps visualize the metrics collected by Prometheus. Here’s how to configure it:

1. **Log in to Grafana:**

   Access Grafana at `http://localhost:3000` using the default credentials (admin/admin).

2. **Add Prometheus as a Data Source:**

   - Navigate to **Configuration** > **Data Sources**.
   - Click **Add Data Source**.
   - Select **Prometheus**.
   - Enter `http://prometheus-stack-kube-prometheus-prometheus.monitoring.svc:9090` as the URL.
   - Click **Save & Test**.

3. **Import Pre-built Dashboards:**

   Grafana provides pre-built dashboards for Kubernetes. You can find and import them from the Grafana website or within Grafana itself.

---
