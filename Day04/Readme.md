To create a detailed article on monitoring custom metrics with Prometheus using a Node.js application on Kubernetes, I'll outline the content in sections to ensure clarity and depth. Here's a detailed step-by-step article:

---

# **Monitoring Custom Metrics in Node.js with Prometheus on Kubernetes**

Monitoring is an essential aspect of application maintenance, especially in a microservices environment. It ensures that developers and system administrators can track the health and performance of services. In this tutorial, we'll walk you through monitoring a simple Node.js microservice with custom metrics using Prometheus in a Kubernetes environment.

## **Overview**

We will:
1. Create a Node.js application with custom metrics using Prometheus.
2. Deploy the application on a Kubernetes cluster.
3. Install Prometheus and configure it to scrape custom metrics from our application.
4. Visualize the data on Prometheus and set up an Alertmanager for notifications.

## **Prerequisites**

- Basic knowledge of Kubernetes and Node.js.
- A working Kubernetes cluster (Minikube or any managed Kubernetes service).
- Helm installed for managing Kubernetes packages.
- kubectl configured to interact with your Kubernetes cluster.
- A Node.js environment for creating and testing the application locally.
  
## **Step 1: Setting Up the Node.js Application with Custom Metrics**

### **1.1 Create a Node.js Application**

Start by setting up a simple Node.js application. Initialize the project with:

```bash
mkdir nodejs-metrics-app
cd nodejs-metrics-app
npm init -y
```

Install the necessary dependencies:

```bash
npm install express prom-client
```

### **1.2 Instrument Metrics with Prometheus Client**

Create an `index.js` file:

```javascript
const express = require('express');
const Prometheus = require('prom-client');

const app = express();
const PORT = process.env.PORT || 3000;

// Initialize the Prometheus registry
const register = new Prometheus.Registry();
Prometheus.collectDefaultMetrics({ register });

// Define custom metrics
const httpRequestCount = new Prometheus.Counter({
  name: 'http_request_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status'],
});

const httpRequestDuration = new Prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.1, 0.5, 1, 2, 5],
});

// Middleware to measure request duration
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    httpRequestCount.inc({ method: req.method, route: req.path, status: res.statusCode });
    end({ method: req.method, route: req.path, status: res.statusCode });
  });
  next();
});

// Sample API endpoints
app.get('/health', (req, res) => {
  res.send('Service is running');
});

app.get('/crash', (req, res) => {
  process.exit(1); // Force the application to crash
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(PORT, () => {
  console.log(`App listening on port ${PORT}`);
});
```

### **1.3 Test Locally**

Start the application locally to ensure everything works:

```bash
node index.js
```

Visit `http://localhost:3000/metrics` to see the Prometheus metrics output.

## **Step 2: Containerize the Application**

Create a `Dockerfile` for the Node.js application:

```dockerfile
# Use a Node.js base image
FROM node:14

# Create app directory
WORKDIR /app

# Install app dependencies
COPY package*.json ./
RUN npm install

# Bundle app source code
COPY . .

# Expose port
EXPOSE 3000

# Start the application
CMD ["node", "index.js"]
```

Build and push the Docker image:

```bash
docker build -t your-dockerhub-username/nodejs-metrics-app .
docker push your-dockerhub-username/nodejs-metrics-app
```

## **Step 3: Deploy the Application on Kubernetes**

### **3.1 Create Kubernetes Manifests**

Create a Kubernetes manifest file `deployment.yaml` to deploy the application:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-metrics-app
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-metrics-app
  template:
    metadata:
      labels:
        app: nodejs-metrics-app
    spec:
      containers:
      - name: nodejs-metrics-container
        image: your-dockerhub-username/nodejs-metrics-app:latest
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-metrics-service
  namespace: dev
spec:
  type: LoadBalancer
  selector:
    app: nodejs-metrics-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
```

Apply the manifest:

```bash
kubectl create namespace dev
kubectl apply -f deployment.yaml
```

### **3.2 Verify Deployment**

Ensure the pods are running:

```bash
kubectl get pods -n dev
```

Check the services:

```bash
kubectl get svc -n dev
```

## **Step 4: Install Prometheus Using Helm**

Install Prometheus using the Helm chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

### **4.1 Expose Prometheus**

Expose Prometheus for local access:

```bash
kubectl port-forward svc/prometheus-stack-kube-prometheus-prometheus -n monitoring 9090:9090
```

Visit `http://localhost:9090` to access the Prometheus UI.

## **Step 5: Configure Service Discovery for Custom Metrics**

### **5.1 Create a ServiceMonitor for Prometheus**

Create a `service-monitor.yaml` file:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nodejs-metrics-monitor
  namespace: dev
spec:
  selector:
    matchLabels:
      app: nodejs-metrics-app
  endpoints:
  - port: 3000
    path: /metrics
    interval: 15s
```

Apply the manifest:

```bash
kubectl apply -f service-monitor.yaml
```

### **5.2 Verify Custom Metrics in Prometheus**

Open Prometheus and query the custom metric `http_request_total`. You should now see data collected from your Node.js application.

## **Step 6: Set Up Alerting with Alertmanager**

### **6.1 Configure Alertmanager**

Create an Alertmanager configuration file:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    route:
      receiver: 'email-alert'
    receivers:
      - name: 'email-alert'
        email_configs:
        - to: 'your-email@example.com'
          from: 'alertmanager@example.com'
          smarthost: 'smtp.example.com:587'
          auth_username: 'your-email@example.com'
          auth_identity: 'your-email@example.com'
          auth_password: 'your-app-password'
```

### **6.2 Deploy the Alertmanager Configuration**

```bash
kubectl apply -f alertmanager-config.yaml
```

### **6.3 Trigger an Alert**

Force a crash in the application:

```bash
kubectl exec -it <nodejs-pod-name> -n dev -- curl http://localhost:3000/crash
```

Check if you receive an alert notification.

## **Conclusion**

In this tutorial, we covered the basics of instrumenting custom metrics in a Node.js application, deploying the application on Kubernetes, and setting up Prometheus to monitor those metrics. Additionally, we configured Alertmanager to receive notifications when something goes wrong. This setup can be further enhanced by integrating a visualization tool like Grafana and exploring advanced metrics.

--- 

Let me know if there's anything else you'd like to include or if you want additional sections!
