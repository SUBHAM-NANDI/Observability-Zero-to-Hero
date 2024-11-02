### Distributed Tracing

When requests flow through a system composed of many microservices, tracing helps observe how each component contributes to the overall processing time of that request. Distributed tracing works by tracking the path and timing of a request across various services, allowing developers to pinpoint where delays or errors might be happening.

#### Why Tracing is Important in Microservices

In a monolithic application, debugging and performance optimization are relatively straightforward since everything is in one place. But in a microservices environment, a single user request may pass through multiple services—each service potentially running on a different server, written in a different language, and managed by a different team. Distributed tracing offers the following benefits:

- **Pinpointing Latency**: When a request takes longer than expected, tracing can show which services or stages in the process are responsible for the delay.
- **Understanding Dependencies**: Tracing maps the dependencies between services, providing visibility into how they interact.
- **Error Tracking**: By capturing each step in a request’s journey, tracing can reveal where errors or failures occur and what dependencies may have contributed.

#### How Tracing Works

To set up tracing, two primary steps are necessary: **instrumenting code** and **deploying tracing infrastructure**.

1. **Instrumentation**:
   - **Instrumentation** is the process of adding code that captures trace data (such as latency and request flow) within the application. This code generates "spans" and assigns them unique IDs to trace requests as they flow between different services.
   - Spans are essential units in tracing; they represent individual operations within a request. For example, a request passing through three services will have three spans, each corresponding to a service.
   - **OpenTelemetry** is a popular, vendor-neutral standard for implementing tracing. It supports various programming languages and integrates well with different tracing backends.

2. **Tracing Infrastructure**:
   - **Jaeger** is an open-source tool often used to collect, store, and visualize trace data.
   - The infrastructure for Jaeger typically involves:
     - **Agent**: Deployed with the application to collect traces. The agent receives trace data from the application and forwards it to the collector.
     - **Collector**: Aggregates traces from multiple agents and processes them.
     - **Storage Backend**: Stores the processed traces. Jaeger supports different storage backends, such as Elasticsearch or Cassandra, for high performance.
     - **User Interface (UI)**: Allows users to query, visualize, and analyze traces.

### An Example Scenario: Implementing Tracing in a Microservices Application

Let’s say you have an e-commerce application built using a microservices architecture, where each major function (login, payment, catalog, etc.) is a separate service. Consider the following request flow:

1. **Login Service** authenticates a user.
2. **Catalog Service** fetches items for the user.
3. **Payment Service** processes the payment.

In this flow, tracing would capture each service's performance and highlight any delays. For instance, if users are experiencing delays in the checkout process, tracing can help you determine if the delay is happening in the payment service, the catalog service, or in the network between these services.

### Jaeger: Architecture and Setup

Jaeger’s architecture is designed to handle and manage traces efficiently in a scalable environment. Here’s a breakdown of its components:

1. **Agent**:
   - The Jaeger agent is typically deployed as a daemon on the same host as the application.
   - It listens for spans emitted by the application’s instrumented code and forwards them to the collector.
   - By placing the agent close to the application, Jaeger reduces network latency for sending trace data.

2. **Collector**:
   - The collector receives traces from the agents and processes them.
   - It aggregates, filters, and prepares the trace data for storage in a database.
   - Collectors can handle high volumes of traces, which is useful in large distributed systems with numerous microservices.

3. **Storage Backend**:
   - Jaeger relies on external storage for long-term data retention and query support.
   - Elasticsearch is commonly used as it allows for fast retrieval of trace data, though other options like Cassandra and Apache Kafka are also supported.

4. **User Interface (UI)**:
   - Jaeger’s UI allows users to view traces, understand request paths, and identify performance bottlenecks.
   - Through the UI, users can visualize how requests flow through the system, view latency at each step, and spot any anomalies.

### Implementing Tracing: Example with OpenTelemetry and Jaeger

Suppose you have a Node.js application with two services: `Service A` and `Service B`. Here’s how you’d implement tracing:

1. **Instrument the Application**:
   - In `Service A` and `Service B`, you would install OpenTelemetry’s SDK.
   - Use OpenTelemetry’s APIs to capture spans. For example:
     ```javascript
     const { trace } = require("@opentelemetry/api");
     const tracer = trace.getTracer("service-a");

     function processRequest() {
       const span = tracer.startSpan("process-request");
       // Perform operations
       span.end();
     }
     ```

2. **Deploy Jaeger on Kubernetes**:
   - You could deploy Jaeger using Helm, a popular Kubernetes package manager, which simplifies installation.
   - Example Helm command:
     ```bash
     helm install jaeger jaegertracing/jaeger
     ```
   - Configure your services to send trace data to the Jaeger agent deployed in the Kubernetes cluster.

3. **Set Up the Storage Backend**:
   - Configure Jaeger to use Elasticsearch for storing traces. This can be specified in the configuration settings.

4. **View and Analyze Traces**:
   - Access Jaeger’s UI to analyze traces. You can view each request’s journey through `Service A` and `Service B` and observe latency at each step.
   - If `Service B` takes longer than expected, you can investigate further by looking at detailed trace data.

### Benefits of Distributed Tracing in Observability

Distributed tracing is invaluable in modern observability as it:

- **Reveals Dependencies**: Shows how services interact and depend on each other.
- **Improves User Experience**: Reduces response times by identifying bottlenecks and optimizing performance.
- **Facilitates Root Cause Analysis**: Enables teams to pinpoint where failures occur in complex distributed systems.

---

Here's a detailed, step-by-step walkthrough of the demo based on the transcript provided. This demo covers setting up observability with Elasticsearch and Jaeger on a Kubernetes (EKS) cluster and integrating the Jaeger tracing tool to visualize application traces.

---

### Step-by-Step Demo for Setting up Jaeger Tracing with Elasticsearch on EKS

#### Prerequisites

1. **Kubernetes Cluster**: Ensure you have a Kubernetes cluster set up on Amazon EKS.
2. **kubectl and eksctl**: Tools for interacting with your EKS cluster.
3. **IAM Role and OIDC**: Ensure that your EKS cluster has OIDC integration enabled for IAM role-based access to EKS.

---

### Part 1: Verify Kubernetes Cluster and Node Configuration

1. **Check Kubernetes Nodes**:
   ```bash
   kubectl get nodes
   ```
   - This command verifies that the EKS cluster and nodes are properly configured and accessible.

2. **Access EKS Cluster ReadMe**:
   - If needed, follow instructions from the Day2 README file to set up an EKS cluster, including adding node groups.

---

### Part 2: Setup Elasticsearch for Log Storage

1. **Create a Service Account for EBS Access**:
   - Create a service account to allow Elasticsearch in EKS to access an EBS volume:
     ```bash
     eksctl create iamserviceaccount \
       --name elasticsearch-sa \
       --namespace logging \
       --cluster <your-cluster-name> \
       --attach-policy-arn arn:aws:iam::aws:policy/AmazonEBSCSIDriverPolicy \
       --approve
     ```
   - This setup includes an IAM role that allows EKS to interact with EBS for storage.

2. **Install EBS CSI Driver**:
   ```bash
   kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=master"
   ```
   - This driver enables EKS to use EBS volumes as persistent storage.

3. **Create a Namespace for Logging**:
   - Separate resources for easier management and debugging:
     ```bash
     kubectl create namespace logging
     ```

4. **Install Elasticsearch via Helm**:
   - Use Helm to install Elasticsearch in the `logging` namespace:
     ```bash
     helm repo add elastic https://helm.elastic.co
     helm install elasticsearch elastic/elasticsearch -n logging
     ```
   - Note down the generated username and password, as they will be used later when configuring Jaeger.

---

### Part 3: Set Up Jaeger for Tracing

1. **Create Namespace for Tracing**:
   - Isolate Jaeger components in a separate namespace:
     ```bash
     kubectl create namespace tracing
     ```

2. **Configure Elasticsearch Certificate**:
   - Extract and save the Elasticsearch certificate to ensure secure communication between Jaeger and Elasticsearch.
   - Create a ConfigMap in Kubernetes to store the certificate:
     ```bash
     kubectl create configmap elasticsearch-cacert --from-file=ca.crt=<path-to-certificate> -n tracing
     ```

3. **Store Elasticsearch Credentials in Kubernetes Secret**:
   - Store Elasticsearch credentials as a Kubernetes secret:
     ```bash
     kubectl create secret generic elasticsearch-credentials \
       --from-literal=username=<your-username> \
       --from-literal=password=<your-password> \
       -n tracing
     ```

4. **Install Jaeger via Helm**:
   - Use Helm to install Jaeger in the `tracing` namespace:
     ```bash
     helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
     helm install jaeger jaegertracing/jaeger -n tracing
     ```
   - Before installation, update the `values.yaml` file with the Elasticsearch credentials:
     - Set the storage username, password, and Elasticsearch URL.

5. **Troubleshoot Jaeger Installation**:
   - If the Jaeger pod is in a crash loop, check for configuration issues, particularly with the liveness and readiness probes. Describe the pod to identify possible issues:
     ```bash
     kubectl describe pod <jaeger-pod-name> -n tracing
     ```
   - If issues are found, double-check the ConfigMap, Secret, and `values.yaml` configurations for errors or missing values.

---

### Part 4: Access Jaeger User Interface

1. **Port Forward Jaeger UI**:
   - Forward the Jaeger UI port to access it locally:
     ```bash
     kubectl port-forward svc/jaeger-query 16686:16686 -n tracing
     ```
   - If accessing from an EC2 instance, add `--address 0.0.0.0` to the port forward command:
     ```bash
     kubectl port-forward svc/jaeger-query 16686:16686 -n tracing --address 0.0.0.0
     ```

2. **Load Jaeger UI in Browser**:
   - Open `http://localhost:16686` in your browser to access Jaeger’s user interface.

---

### Part 5: Deploy a Sample Application with Tracing

1. **Deploy Instrumented Applications**:
   - Download and deploy a sample application (Service A and Service B) instrumented with OpenTelemetry:
     ```bash
     kubectl apply -k <path-to-manifest>/day4
     ```
   - Verify that the applications have been deployed:
     ```bash
     kubectl get pods -n <namespace>
     ```

2. **Trigger Application Endpoints**:
   - Test application endpoints to generate traces that can be viewed in Jaeger:
     ```bash
     curl http://<load-balancer-url>/healthy
     ```
   - Other endpoints (like `/serviceB` or `/serverError`) can be used to generate different traces.

---

### Part 6: View Traces in Jaeger

1. **Locate Traces in Jaeger UI**:
   - Go to the Jaeger UI, select the application service (e.g., Service A), and click “Find Traces.”
   - Examine traces and spans, which represent points in the journey of each request.
   - Each span shows specific operation details, including duration, which can be used to identify performance bottlenecks.

2. **Analyze Trace Details**:
   - Click on individual spans to view deeper information, such as:
     - Start and end times
     - Time taken by each operation
   - Use this data to identify any latency issues, allowing you to pinpoint and optimize inefficient code or configuration.

---

### Additional Tips for Optimization

1. **Adjust Probes**:
   - If liveness or readiness probes are causing issues, adjust the timeouts or parameters in the Helm `values.yaml`.

2. **Namespace Organization**:
   - Separate namespaces for logging, monitoring, and tracing make it easier to troubleshoot and manage access controls (RBAC).

3. **Load Balancer and Security**:
   - Consider exposing Jaeger via a Kubernetes Ingress or LoadBalancer with appropriate security settings for production environments.

---
