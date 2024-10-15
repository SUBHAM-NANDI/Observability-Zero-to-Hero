Link - https://github.com/iam-veeramalla/observability-zero-to-hero/tree/main/day-2

### **Step 1: Set Up the Prerequisites**

Before creating the EKS cluster, you need to install and configure some essential tools.

#### **1.1 Install AWS CLI**

- Download and install the AWS CLI by following [AWS CLI installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

#### **1.2 Configure AWS CLI**

After installation, configure AWS CLI using the following command. This sets up your AWS credentials.
```bash
aws configure
```
You will be prompted to input your AWS Access Key ID, Secret Access Key, region, and output format.

#### **1.3 Install eksctl**

- Install `eksctl` using the steps mentioned in the [eksctl installation guide](https://eksctl.io/introduction/#installation).

#### **1.4 Install kubectl**

- Install `kubectl` by following the instructions in the [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

---

### **Step 2: Create an EKS Cluster**

Now that you have the tools installed, you can create your EKS cluster.

#### **2.1 Create EKS Cluster Without Node Group**
```bash
eksctl create cluster --name=observability \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup
```
- This command creates a new EKS cluster named `observability` in the `us-east-1` region, across two availability zones: `us-east-1a` and `us-east-1b`.
- The `--without-nodegroup` option creates the cluster without any worker nodes. We'll add a node group next.

#### **2.2 Associate IAM OIDC Provider**
```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster observability \
    --approve
```
This step enables IAM roles for Kubernetes service accounts, a necessary step for using AWS service integrations like ECR and ALB.

#### **2.3 Create a Managed Node Group**
```bash
eksctl create nodegroup --cluster=observability \
                        --region=us-east-1 \
                        --name=observability-ng-private \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=3 \
                        --node-volume-size=20 \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking
```
- **Cluster:** Adds a node group to the `observability` EKS cluster.
- **Node Group Configuration:** This node group has 2–3 `t3.medium` EC2 instances, with 20 GB of storage per node, and it will be deployed in a private subnet.
- **Access Options:** Includes permissions for Auto Scaling (ASG), External DNS, ECR (container registry), AWS App Mesh, and ALB Ingress.

#### **2.4 Update the Kubernetes Config**
```bash
aws eks update-kubeconfig --name observability
```
This command updates the local `kubeconfig` file with the credentials and configuration for accessing the newly created EKS cluster.

---

### **Step 3: Install the kube-prometheus-stack Using Helm**

Next, we will install Prometheus and Grafana for monitoring, using Helm.

#### **3.1 Add the Prometheus Community Helm Chart**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
This adds the Prometheus Helm chart repository and updates it to ensure you have access to the latest charts.

#### **3.2 Create a Namespace for Monitoring**
```bash
kubectl create ns monitoring
```
A separate namespace for monitoring ensures a clean and isolated environment for your Prometheus setup.

#### **3.3 Install the kube-prometheus-stack Chart**
```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
-f ./custom_kube_prometheus_stack.yml
```
- **Chart:** Installs the `kube-prometheus-stack` chart in the `monitoring` namespace.
- **Custom Values:** The `-f ./custom_kube_prometheus_stack.yml` flag applies any custom configurations you’ve specified in the YAML file.

---

### **Step 4: Verify the Installation**

After installing Prometheus and Grafana, you can verify their status and access their interfaces.

#### **4.1 Check All Resources in the Monitoring Namespace**
```bash
kubectl get all -n monitoring
```
This command lists all resources (Pods, Services, Deployments, etc.) created in the `monitoring` namespace to confirm they are up and running.

#### **4.2 Access the Prometheus UI**
```bash
kubectl port-forward service/prometheus-operated -n monitoring 9090:9090
```
- **Port Forwarding:** This forwards local port `9090` to Prometheus’ web UI. You can access it by navigating to `http://localhost:9090` in your browser.
- **PromQL:** The Prometheus web interface allows you to run ad-hoc queries using the PromQL language.

#### **4.3 Access the Grafana UI**
```bash
kubectl port-forward service/monitoring-grafana -n monitoring 8080:80
```
- **Port Forwarding:** Forwarding port `8080` locally will give you access to Grafana’s dashboard. You can access Grafana by navigating to `http://localhost:8080`.
- **Login Details:** The default username is `admin` and the default password is `prom-operator`.

#### **4.4 Access the Alertmanager UI**
```bash
kubectl port-forward service/alertmanager-operated -n monitoring 9093:9093
```
- **Port Forwarding:** This will allow you to access Alertmanager’s UI via `http://localhost:9093`.

---

### **Step 5: Clean Up Resources**

After completing your experiments or if you no longer need the monitoring setup, clean up the resources.

#### **5.1 Uninstall the Prometheus Helm Chart**
```bash
helm uninstall monitoring --namespace monitoring
```
This will uninstall the `kube-prometheus-stack` from the `monitoring` namespace.

#### **5.2 Delete the Monitoring Namespace**
```bash
kubectl delete ns monitoring
```
This removes the `monitoring` namespace and any remaining resources within it.

#### **5.3 Delete the EKS Cluster**
```bash
eksctl delete cluster --name observability
```
This will delete the EKS cluster and all associated resources.

---
