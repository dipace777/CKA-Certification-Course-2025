# Day 49: MASTER Kubernetes Ingress | PART 1 | Path-Based Routing Demo on Multi-AZ Amazon EKS with AWS ALB

## Video reference for Day 49 is the following:



---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

---

## Table of Contents

- [Introduction](#introduction)  
- [Why Ingress?](#why-ingress)  
- [What Is Ingress?](#what-is-ingress)  
- [What is Ingress Controller](#ingress-controller)   
- [Demo 1: Path-Based Routing Using Ingress on Amazon EKS](#demo-1-path-based-routing-using-ingress-on-amazon-eks)
  - [What Are We Going to Deploy?](#what-are-we-going-to-deploy) 
  * [Cluster Configuration](#cluster-configuration)
  * [Prerequisites](#prerequisites)
  * [Step 1: Provision EKS Cluster](#step-1-provision-eks-cluster)
  * [Step 2: Install AWS Load Balancer Controller](#step-2-install-aws-load-balancer-controller)
    * [2.1 Create IAM Policy](#21-create-iam-policy)
    * [2.2 Create an IAM-Backed Kubernetes Service Account](#22-create-an-iam-backed-kubernetes-service-account)
    * [2.3 Install the AWS Load Balancer Controller Using Helm](#23-install-the-aws-load-balancer-controller-using-helm)
  * [Step 3: Create Deployments and Services](#step-3-create-deployments-and-services)
    * [3.1 Create a Dedicated Namespace](#31-create-a-dedicated-namespace)
    * [3.2 iPhone Deployment and Service](#32-iphone-deployment-and-service)
    * [3.3 Android Deployment and Service](#33-android-deployment-and-service)
    * [3.4 Desktop Deployment and Service](#34-desktop-deployment-and-service)
  * [Step 4: Creating the Ingress Resource](#step-4-creating-the-ingress-resource)
    * [Understanding ALB Target Types in Amazon EKS](#understanding-alb-target-types-in-amazon-eks)
      * [`target-type: ip`](#target-type-ip--direct-to-pod-routing-via-vpc-cni)
      * [`target-type: instance`](#target-type-instance--nodeport-based-routing-via-kube-proxy)
    * [Understanding `IngressClass`](#understanding-ingressclass)
  * [Step 5: End-to-End Verification](#step-5-end-to-end-verification)
    * [Request Flow](#request-flow-how-traffic-reaches-the-pods-with-target-type--ip)
    * [Test the Routing](#test-the-routing)
    * [Verify the Ingress Configuration](#verify-the-ingress-configuration)
    * [Verify from the AWS Console](#verify-from-the-aws-console)
  * [Step 6: Cleanup](#step-6-cleanup)  
- [Conclusion](#conclusion)  
- [References](#references)  

---


## Introduction

Ingress in Kubernetes is a powerful API object that enables centralized, fine-grained control over HTTP and HTTPS routing to services within your cluster. Rather than exposing each service individually via NodePorts or LoadBalancers, Ingress consolidates external access through a single, manageable entry point.

This is **Part 1 of a two-part Ingress series** on Amazon EKS, where we’ll implement a production-grade setup using the **AWS Load Balancer Controller (ALB)** across a multi-AZ Kubernetes cluster. In this first part, we focus on:

* Understanding how Kubernetes Ingress works under the hood.
* Deploying a Multi-AZ EKS cluster tailored for Ingress use-cases.
* Implementing **path-based routing** to serve multiple applications using a **public AWS ALB**.
* Verifying pod-level traffic routing using `target-type: ip`.

In **Part 2 (Day 50)**, we’ll extend this foundation by:

* Registering a **custom domain** using Route 53.
* Enabling **TLS termination** using AWS Certificate Manager (ACM).
* Implementing **host-based routing** using subdomains (e.g., `iphone.example.com`, `android.example.com`) over the same ALB.

Whether you're preparing for real-world deployments or pursuing the CKA exam, this series will equip you with a hands-on, cloud-native understanding of Ingress on Amazon EKS using best practices and native AWS integrations.

---

## Why Ingress?

You can expose an application externally using a `NodePort` service, but as discussed in our [Services lecture (Day 12)](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2012), this method does not scale well. In a real-world Kubernetes cluster with auto-scaling enabled, nodes come and go frequently. Sharing external IPs of dynamic nodes with users or external systems becomes impractical. While `NodePort` is acceptable for dev and test, it is not ideal for production workloads.

A more common option is the `LoadBalancer` service type, which provisions a cloud-managed load balancer (like AWS ELB) to route traffic to your service. This solves some issues but introduces others:

* It provisions a **Layer 4 (TCP/UDP) load balancer** by default, lacking HTTP-level (Layer 7) awareness. This means no path- or host-based routing out of the box.
* Each `LoadBalancer` service creates a **separate cloud load balancer**, incurring hourly charges. This becomes expensive and hard to manage when scaling out microservices.
* **TLS termination must be handled by the application**, since L4 load balancers can’t decrypt traffic. This burdens your app with certificate management and TLS logic, which is better handled at the edge.


## What Are We Looking For?

Ideally, we want a **Kubernetes-native solution** that supports:

* HTTP-aware routing (e.g., `/android`, `/ios`, `/`, or subdomains like `android.myapp.com`)
* A **single Load Balancer** serving multiple apps — reducing cost and operational complexity
* Declarative routing rules, version-controlled and managed like any other Kubernetes object

This is where **Ingress** comes in.

---

## What Is Ingress?

Ingress is a **Kubernetes API object** that defines **rules for routing external HTTP(S) traffic** to internal services in your cluster. Think of it as a **Layer 7 HTTP(S) router**—mapping requests based on **URL path**, **host name**, or **TLS configuration** to the appropriate **Kubernetes Service**.

Ingress enables:

* **Path-based routing** (`/android` → `android-svc`)
* **Host-based routing** (`iphone.myapp.com` → `ios-svc`)
* **TLS termination** (offloading certificate handling from apps)
* **A single, centralized entry point** for all HTTP(S) traffic

However, an `Ingress` resource is **not a traffic hop or runtime network component**—it’s just a **declarative YAML configuration**. On its own, it does nothing.

> ⚠️ An Ingress resource **needs an Ingress Controller** running in your cluster to interpret and implement the routing rules.

---

## Ingress Controller

An **Ingress Controller (IC)** is a Kubernetes component that watches for `Ingress` resources and **translates their rules into actual load balancer configurations**.

In simple terms:

> The Ingress resource defines the **“what”** (rules). The Ingress Controller handles the **“how”** by provisioning and configuring the cloud **HTTP(S) Load Balancer** to match those rules.

Most Ingress Controllers configure an external **Layer 7 Load Balancer**, but in Kubernetes the only Layer 7 protocols supported are **HTTP and HTTPS**. So, while the term "Layer 7" is often used, **“HTTP(S) Load Balancer”** is a more accurate description.

---

### Let’s Walk Through a Real-World Example

![Alt text](/images/49a.png)

To make this concrete, consider the following flow:

👩‍💻 **Shwetangi** visits `myapp.com/iphone`.
Here's what happens:

1. Her request travels through the internet to a **cloud-managed HTTP(S) Load Balancer**, such as an AWS ALB.
2. This Load Balancer was **provisioned by the Ingress Controller** running inside the Kubernetes cluster.
3. The Load Balancer was configured using the rules defined in the **Ingress Resource** (e.g., path `/iphone` should go to `iphone-svc`).
4. The request is forwarded to the **Kubernetes Service (`iphone-svc`)**, which then sends it to a matching **Pod**.

> 📌 The **Ingress Resource is not an intermediate hop**. It exists only to define rules. The **Ingress Controller reads these rules and implements them at the Load Balancer level**.

The diagram includes the Ingress resource for clarity, but the routing happens **at the Load Balancer**, not inside Kubernetes.

---

### Example: AWS Load Balancer Controller (ALB on Amazon EKS)

Here’s what happens under the hood:

1. You create an Ingress object that says:
   `/iphone → iphone-svc`
2. The **AWS Load Balancer Controller** running in your cluster detects the new Ingress.
3. It provisions an **AWS ALB** and configures:

   * **Listeners** (on ports 80 and 443)
   * **Target Groups** pointing to Kubernetes Pods (via IPs or NodePorts)
   * **Routing rules** based on host/path/TLS settings in the Ingress

It **continuously syncs** the ALB config with your Ingress definition. So if you update paths, hosts, or TLS certs, those changes are reflected in real-time.

---

### ⚠️ Important AWS-Specific Note

Since we used AWS as our example, you should understand:

* The **AWS Load Balancer Controller (LBC)** can provision:

  * **ALB** (Application Load Balancer for HTTP/S) when you use an **Ingress resource**
  * **NLB** (Network Load Balancer for TCP/UDP) when you define a **Service of type `LoadBalancer`** with specific annotations

This makes the AWS LBC a **dual-purpose controller**, supporting both **L4 and L7 traffic patterns** based on how the resource is defined.

And while the `Ingress` spec is standardized, **each controller** (AWS, NGINX, HAProxy, etc.) supports **provider-specific features** via **annotations**.
These annotations let you:

* Set SSL redirect rules
* Change health check behavior
* Control session affinity, timeout, or request rewrites

> 📘 To explore the full capabilities of your controller, refer to its official documentation:
> 👉 [AWS Load Balancer Controller Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)

---

### Popular Ingress Controllers

| Controller                       | Description                                                                                 |
| -------------------------------- | ------------------------------------------------------------------------------------------- |
| **AWS Load Balancer Controller** | Manages ALBs and NLBs dynamically on EKS. Deep AWS integration with high scalability        |
| **NGINX Ingress Controller**     | Most widely used, highly customizable with strong community support                         |
| **HAProxy Ingress**              | Optimized for high performance and low latency routing                                      |
| **Traefik**                      | Cloud-native, supports automatic TLS via Let’s Encrypt and real-time dynamic config updates |
| **Contour**                      | Built on Envoy, offers advanced traffic control and multi-team ingress management           |
| **Istio Ingress Gateway**        | Part of the Istio service mesh. Enables policies, mTLS, and fine-grained traffic management |

> All of these implement the same Kubernetes `Ingress` API—but their **operational behavior, ecosystem fit, and feature sets vary**.
> Choose the one that best aligns with your **platform, scale, and operational model**.

---


## What’s Ahead

Ingress is a cornerstone of production-grade Kubernetes networking. By mastering Ingress, you unlock **advanced HTTP(S) routing**, **TLS offloading**, and **cost-effective traffic management** across services. In this 2-part Ingress deep dive, we’ll move from foundational concepts to production-ready architectures on Amazon EKS.

* **Demo 1: Path-Based Routing on Multi-AZ Amazon EKS**
  Covered in **Day 49**, we begin with Ingress using the **AWS Load Balancer Controller**, deploying three apps (`/iphone`, `/android`, `/`) behind an **Application Load Balancer** (ALB) in a real-world EKS cluster.

* **Demo 2: TLS with ACM and Custom Domain via Route 53**
  Coming up in **Day 50**, we’ll secure the setup with **HTTPS**, leveraging **AWS Certificate Manager (ACM)** and a domain managed through **Route 53**.

* **Demo 3: Name-Based Routing with AWS ALB**
  Also in **Day 50**, we’ll route requests like `iphone.cwvj.click` and `android.cwvj.click` to the correct services using **hostname-based Ingress rules**, all via the same ALB.

Let’s get started with **Demo 1 in Day 49**, and watch Ingress in action on Amazon EKS.

---

## Demo 1: Path-Based Routing Using Ingress on Amazon EKS

In this demo, we’ll deploy a lightweight multi-service application named `app1` in the namespace `app1-ns` on a **Multi-AZ Amazon EKS cluster**. This application simulates three **frontend microservices** — one for iPhone users, one for Android, and a default desktop fallback — and exposes them externally using **Kubernetes Ingress** backed by an **AWS Application Load Balancer (ALB)**.

This demo focuses on **path-based routing**, where requests are routed to different services based on the URL path. It's a production-ready pattern that enables simplified routing and centralized traffic management using native Kubernetes constructs and AWS integration.

---

### What Are We Going to Deploy?

![Alt text](/images/49c.png)

Based on the architecture diagram, here’s what the deployment involves:

1. **Three frontend microservices** under the `app1` namespace:

   * `iphone-svc` → serves requests to `/iphone`
   * `android-svc` → serves requests to `/android`
   * `desktop-svc` → serves as the default catch-all for `/` and unmatched paths

2. An **Ingress Resource** defines the routing logic, mapping URL paths to the corresponding Kubernetes Services.

3. The **AWS Load Balancer Controller (ALB Ingress Controller)** continuously watches for changes in Ingress resources and:

   * Provisions an **external HTTP(S) Load Balancer** in your AWS account
   * Creates routing rules and target groups based on the Ingress definition

4. **Traffic flow:**

   * User (e.g., Shwetangi) accesses `<lb-dnsname>/iphone` via the Internet.
   * The ALB receives the request and applies the listener rules based on path.
   * Traffic is routed directly to the corresponding Pods (on port `5678`), bypassing the NodePort layer using **target-type: ip** (enabled by the VPC CNI plugin).

5. All Kubernetes components — Services, Pods, and Ingress Controller — are deployed across **two Availability Zones**, offering fault tolerance and high availability.

> **Note:** The Ingress resource is declarative and doesn’t process traffic itself. It simply informs the Ingress Controller how the Load Balancer should be configured.

---

### **Cluster Configuration**

![Alt text](/images/49b.png)
This demo is built on an **Amazon EKS cluster** deployed in the `us-east-2 (Ohio)` region. The cluster spans two Availability Zones: `us-east-2a` and `us-east-2b`. It includes **four worker nodes** based on `t3.small` EC2 instances.

* **Why 4 nodes?** This demo uses 4 worker nodes to showcase features like `topologySpreadConstraints`, which rely on sufficient node distribution across AZs. However, you can reduce this to 2 for basic Ingress setups.
* **Why t3.small?** This instance type is eligible under AWS’s `$100 free tier credits`, making it ideal for learning environments. When provisioning via the AWS Console, non-eligible instance types appear greyed out, simplifying selection.

---

### **Prerequisites**

#### 1. Understanding of Kubernetes Services

Before beginning, ensure you’re comfortable with Kubernetes Services. These concepts were covered in Day 12 of the course:

* [YouTube Lecture (Day 12)](https://www.youtube.com/watch?v=92NB8oQBtnc&ab_channel=CloudWithVarJosh)
* [GitHub Resources (Day 12)](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2012)

#### 2. Install Required Tools

Ensure the following tools are installed on your local machine or cloud jump-host:

* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [eksctl](https://eksctl.io/installation/)
* [helm](https://helm.sh/docs/intro/install/)

---

## **Step 1: Provision EKS Cluster**

Create a file named `eks-config.yaml` with the following contents:

```yaml
apiVersion: eksctl.io/v1alpha5       # Defines the API version used by eksctl for parsing this config
kind: ClusterConfig                  # Declares the type of resource (an EKS cluster configuration)

metadata:
  name: cwvj-ingress-demo            # Name of the EKS cluster to be created
  region: us-east-2                 # AWS region where the cluster will be deployed (Ohio)
  tags:                             # Custom tags for AWS resources created as part of this cluster
    owner: varun-joshi              # Tag indicating the owner of the cluster
    bu: cwvj                        # Tag indicating business unit or project group
    project: ingress-demo           # Tag for grouping resources under the ingress demo project

availabilityZones:
  - us-east-2a                      # First availability zone for high availability
  - us-east-2b                      # Second availability zone to span the cluster

iam:
  withOIDC: true                    # Enables IAM OIDC provider, required for IAM roles for service accounts (IRSA)

managedNodeGroups:
  - name: cwvj-eks-priv-ng          # Name of the managed node group
    instanceType: t3.small          # EC2 instance type for worker nodes
    minSize: 4                      # Minimum number of nodes in the group
    maxSize: 4                      # Maximum number of nodes (fixed at 4 here; no autoscaling)
    privateNetworking: true         # Launch nodes in **private subnets only** (no public IPs)
    volumeSize: 20                  # Size (in GB) of EBS volume attached to each node
    iam:
      withAddonPolicies:           # Enables AWS-managed IAM policies for certain addons
        autoScaler: true           # Allows Cluster Autoscaler to manage this node group
        externalDNS: false         # Disables permissions for ExternalDNS (not used in this demo)
        certManager: yes           # Grants cert-manager access to manage certificates using IAM
        ebs: false                 # Disables EBS volume policy (not required here)
        fsx: false                 # Disables FSx access
        efs: false                 # Disables EFS access
        albIngress: true           # Grants permissions needed by AWS Load Balancer Controller (ALB)
        xRay: false                # Disables AWS X-Ray (tracing not needed)
        cloudWatch: false          # Disables CloudWatch logging from nodes (optional in minimal setups)
    labels:
      lifecycle: ec2-autoscaler     # Custom label applied to all nodes in this group (useful for targeting in node selectors or autoscaler configs)

```

Create the cluster using:

```bash
eksctl create cluster -f eks-config.yaml
```

Verify the node distribution across AZs:

```bash
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone
```

This ensures that the nodes are evenly distributed across the specified availability zones, which is critical for demonstrating topology-aware workloads.

---

## **Step 2: Install AWS Load Balancer Controller**

The AWS Load Balancer Controller is required for managing ALBs (Application Load Balancers) in Kubernetes using the native `Ingress` API. It watches for `Ingress` resources and creates ALBs accordingly, enabling advanced HTTP features such as path-based and host-based routing. It also supports integration with target groups, health checks, SSL termination, and more.

This step involves two sub-phases:

* Setting up the IAM permissions that allow the controller to interact with AWS APIs
* Installing the controller itself using Helm

---

### **2.1 Create IAM Policy**

The AWS Load Balancer Controller requires specific IAM permissions to provision and manage ALB resources on your behalf (such as creating Target Groups, Listeners, and Rules).

Download the required IAM policy:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json
```

Create the policy in your AWS account:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

> The created policy will be used to grant your controller the ability to call ELB, EC2, and IAM APIs. You can inspect the JSON file for exact permissions.

---

### **2.2 Create an IAM-Backed Kubernetes Service Account**

We now create a Kubernetes `ServiceAccount` that is linked to the IAM policy created above. This is achieved using `eksctl`, which automatically sets up the necessary IAM Role and CloudFormation resources under the hood.

```bash
eksctl create iamserviceaccount \
  --cluster=cwvj-ingress-demo \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::261358761470:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-2 \
  --approve
```

This command does the following:

* Creates an **IAM Role** with the required policy attached (visible in AWS CloudFormation).
* Annotates a Kubernetes **ServiceAccount** with this role.
* Binds the ServiceAccount to the controller pods that we will deploy in the next step.

To verify:

```bash
kubectl get sa -n kube-system aws-load-balancer-controller -o yaml
```

Example output:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::261358761470:role/eksctl-cwvj-ingress-demo-addon-iamserviceacco-Role1-Llblca1iSsNh
```

This annotation allows the controller pod to **assume the IAM role** and make API calls securely from within the cluster.

---

### **2.3 Install the AWS Load Balancer Controller Using Helm**

> Note: The **AWS Load Balancer Controller** was previously known as the **ALB Ingress Controller**. While the functionality has expanded beyond ALBs, many community articles and annotations still use the older term. The official and recommended name is now AWS Load Balancer Controller.

Add the AWS-maintained Helm chart repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Install the controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=cwvj-ingress-demo \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.13.0
```

> The `--set serviceAccount.create=false` flag ensures that Helm does not attempt to create a new service account. We are using the one we created and annotated earlier using `eksctl`.

> `helm list` alone won’t show the AWS Load Balancer Controller, as it’s installed in the `kube-system` namespace. Use `helm list -n kube-system` or `helm list -A` to view it.


Optional: List available versions of the chart

```bash
helm search repo eks/aws-load-balancer-controller --versions
```

Verify that the controller is installed:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Expected output:

```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           84s
```

Inspect the pods to verify the correct service account is mounted:

```bash
kubectl describe pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

Look for this line under the pod description:

```
Service Account: aws-load-balancer-controller
```

This confirms the pods are using the IAM-bound service account, enabling them to create ALBs, target groups, and associated rules when `Ingress` resources are created.

---
**Reference Links**

* **AWS Documentation**: [Installing AWS Load Balancer Controller](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)
* **GitHub Repository**: [AWS Load Balancer Controller GitHub](https://github.com/kubernetes-sigs/aws-load-balancer-controller)


---

## Step 3: Create Deployments and Services for iPhone, Android, and Desktop Users

This step creates three separate deployments and corresponding services, each exposing a static HTML page to simulate platform-specific landing pages. These applications will be accessed via context path routing using an Ingress resource in the later steps.

---

### **3.1 Create a Dedicated Namespace**

**01-ns.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
```

> We create a dedicated namespace `app1-ns` to logically isolate all resources related to this application. Namespaces help with resource scoping, management, and RBAC.

Apply the namespace:

```bash
kubectl apply -f 01-ns.yaml
```

Set it as the default for the current context to avoid specifying `-n app1-ns` repeatedly:

```bash
kubectl config set-context --current --namespace=app1-ns
```

---

### **3.2 iPhone Deployment and Service**

**02-iphone.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iphone-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: iphone-page
  template:
    metadata:
      labels:
        app: iphone-page
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: iphone-page
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            mkdir -p /iphone && echo '<html>
              <head><title>iPhone Users</title></head>
              <body>
                <h1>iPhone Users</h1>
                <p>Welcome to Cloud With VarJosh</p>
              </body>
            </html>' > /iphone/index.html && cd / && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: iphone-svc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /iphone/index.html
spec:
  selector:
    app: iphone-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

**Explanation:**

* `python:alpine` image runs a minimal HTTP server on port `5678` to serve `/iphone/index.html`.
* `topologySpreadConstraints` ensure pods are distributed across availability zones (`topology.kubernetes.io/zone`) with `maxSkew: 1`, minimizing zonal skew.
* The Service listens on port 80 (ClusterIP), forwarding to container port 5678.
* ALB health check is scoped to `/iphone/index.html` via annotation on the service.

Apply the resources:

```bash
kubectl apply -f 02-iphone.yaml
```

---

### **3.3 Android Deployment and Service**

**03-android.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: android-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: android-page
  template:
    metadata:
      labels:
        app: android-page
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: android-page
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            mkdir -p /android && echo '<html>
              <head><title>Android Users</title></head>
              <body>
                <h1>Android Users</h1>
                <p>Welcome to Cloud With VarJosh</p>
              </body>
            </html>' > /android/index.html && cd / && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: android-svc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /android/index.html
spec:
  selector:
    app: android-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

**Explanation:**

* Follows the same pattern as iPhone, except the route is `/android/index.html`.
* Health checks are isolated per app context, which is crucial when a single ALB serves multiple routes.
* Distribution of pods across zones is maintained.

Apply the resources:

```bash
kubectl apply -f 03-android.yaml
```

---

### **3.4 Desktop Deployment and Service**

**04-desktop.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: desktop-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: desktop-page
  template:
    metadata:
      labels:
        app: desktop-page
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: desktop-page
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo '<html>
              <head><title>Desktop Users</title></head>
              <body>
                <h1>Desktop Users</h1>
                <p>Welcome to Cloud With VarJosh</p>
              </body>
            </html>' > /index.html && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: desktop-svc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /index.html
spec:
  selector:
    app: desktop-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

**Explanation:**

* Desktop variant serves content at `/index.html` from the root path, suitable for default catch-all routing.
* The rest of the configuration follows the same pattern.
* Health check path aligns with this root content.

Apply the resources:

```bash
kubectl apply -f 04-desktop.yaml
```

---

### **Verification Commands**

To verify deployments and services:

```bash
kubectl get deployments
kubectl get services
```

To verify pod distribution across AZs:

```bash
kubectl get pods -o wide --sort-by='.spec.nodeName'
```

You’ll observe that pods from each deployment are spread across both AZs, thanks to the `topologySpreadConstraints`.

---

## **Step 4: Creating the Ingress Resource**

When you apply an Ingress resource in a Kubernetes cluster configured with the **AWS Load Balancer Controller**, it acts as a blueprint. The controller watches for Ingress objects and, upon detecting this configuration, **provisions an actual AWS Application Load Balancer (ALB)** using the specifications defined in the manifest.

> Think of the Ingress resource not as a traffic hop but as a **declarative configuration** for the external ALB that will be created in AWS. The traffic still flows directly through the ALB to your service pods.

Since the AWS Load Balancer Controller is installed and mapped to a service account with the necessary IAM permissions, it can interact with AWS APIs to automatically create:

* An ALB with listeners
* Target groups for your services
* Health check settings
* Routing rules

### **Manifest: 05-ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1           # API version for Ingress resource; stable since Kubernetes v1.19
kind: Ingress                              # Declares that this manifest defines an Ingress resource
metadata:
  name: ingress-demo1                      # Name of the Ingress resource
  namespace: app1-ns                       # Namespace in which this Ingress is deployed
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Makes the ALB internet-facing (publicly accessible); other option is 'internal' for private use

    alb.ingress.kubernetes.io/load-balancer-name: cwvj-ingress-demo1
    # Assigns a custom name to the ALB; helpful for identification in the AWS Console

    alb.ingress.kubernetes.io/target-type: ip
    # Registers individual Pod IPs (via VPC CNI) in ALB target groups, skipping NodePort/host networking

    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    # Configures the ALB to listen on HTTP port 80; JSON array allows multiple listeners if needed (e.g., HTTPS)

    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    # Defines protocol used for ALB health checks (can be HTTP or HTTPS)

    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    # Uses the same port as the listener for health checks ('traffic-port' is a special keyword)

    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    # Frequency (in seconds) with which ALB sends health check requests

    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    # Max time (in seconds) to wait for a health check response

    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    # Number of successful health checks before a target is marked healthy

    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    # Number of failed health checks before a target is marked unhealthy

    alb.ingress.kubernetes.io/success-codes: '200'
    # ALB considers HTTP 200 as a successful health check response

spec:
  ingressClassName: alb                    # Instructs Kubernetes to use the AWS ALB Ingress Controller for this resource

  rules:                                   # List of rules that define how traffic is routed
    - http:                                # All rules here apply to HTTP traffic
        paths:                             # Each entry defines a routing path
          - path: /iphone
            pathType: Prefix               # Match all paths that start with '/iphone'
            backend:
              service:
                name: iphone-svc           # Traffic matching '/iphone' goes to this Kubernetes Service
                port:
                  number: 80               # Service port (which maps to containerPort 5678 inside pods)

          - path: /android
            pathType: Prefix               # Match all paths starting with '/android'
            backend:
              service:
                name: android-svc          # Routes traffic to android-svc
                port:
                  number: 80               # Matches the port exposed by the android-svc

          - path: /
            pathType: Prefix               # Catch-all path for requests that don't match the above
            backend:
              service:
                name: desktop-svc          # Default backend service for unmatched paths (e.g., desktop)
                port:
                  number: 80               # Service port for desktop-svc

```

Apply the resource:

```bash
kubectl apply -f 05-ingress.yaml
```

---


## Understanding ALB Target Types in Amazon EKS

When you use the **AWS Load Balancer Controller** with Kubernetes `Ingress`, the annotation `alb.ingress.kubernetes.io/target-type` controls how backend targets are registered in the ALB Target Group. It defines how traffic flows from the ALB to your Pods.

In **Demo 1**, you're performing **path-based routing** using an ALB with `/iphone`, `/android`, and `/` as prefixes. DNS, Route 53, and TLS via ACM are not introduced yet — those come in later demos.

---

### `target-type: ip` — Direct-to-Pod Routing via VPC CNI
![Alt text](/images/49d.png)
When you set `target-type: ip`, the ALB registers **Pod IPs** directly in its Target Groups. This is possible because the **AWS VPC CNI plugin** assigns each Pod a routable secondary IP from the VPC subnet range. The Kubernetes `Service` is still needed, but only for discovering the backend Pods — it does not participate in traffic forwarding.

> Reduces network hops and supports Pod-level health checks.

**Flow (Left to Right):**

```
Shwetangi → <lb-dnsname>/iphone → Internet → AWS ALB → iphone target group (type = ip) → Pod IP (via Node’s VPC ENI) → iphone container (port 5678)
```

#### Key Pointers (as per diagram):

* **Kubernetes Service** is used **only** to identify backend Pods.
* **ALB forwards traffic directly** to Pod IPs (fewer hops).
* **Requires correct Security Group rules on Node ENIs**, which are **automatically handled by the AWS Load Balancer Controller** during ALB provisioning.

Suitable for clusters using **AWS VPC CNI** (default on EKS).

---

### `target-type: instance` — NodePort-Based Routing via kube-proxy
![Alt text](/images/49e.png)
When using `target-type: instance`, the ALB registers **EC2 worker nodes** (not Pod IPs) in the Target Group. For this to work, the backing Kubernetes Service must be of type **NodePort**, which exposes a high-range port on each node. The ALB sends traffic to these NodePorts. Then, the traffic flows through **`kube-proxy`** to reach a matching Pod using internal routing.

> ⚠️ Adds more hops and only supports node-level health checks.

**Flow (Left to Right):**

```
Shwetangi → <lb-dnsname>/iphone → Internet → AWS ALB → iphone target group (type = instance) → EC2 Node (NodePort) → kube-proxy → ClusterIP Service → Pod IP → iphone container (port 5678)
```

#### Key Pointers (as per diagram):

* **ALB routes to EC2 NodePort**; traffic flows through **kube-proxy**.
* Adds **extra network hops** and indirect routing.
* **Works without VPC CNI**, making it ideal for alternate CNIs like **Calico**, **Weave**, etc.

This mode is more flexible in terms of CNI support, but slightly less efficient.

---


### **Understanding `IngressClass`**

Just like Kubernetes uses `StorageClass` to dynamically provision different types of volumes (e.g., gp3, io1), `IngressClass` enables the selection of a specific ingress controller. This is important when:

* Your cluster uses multiple ingress controllers (e.g., AWS ALB, NGINX, Istio)
* You want to default routing to a particular controller when none is explicitly mentioned

Check available ingress classes:

```bash
kubectl get ingressclass
```

Sample output:

```
NAME   CONTROLLER            PARAMETERS   AGE
alb    ingress.k8s.aws/alb   <none>       3h38m
```

In our case, `alb` was auto-created during AWS LBC installation.

---
Check the deployed ingress:

```bash
kubectl get ingress
```

Sample output:

```
NAME            CLASS   HOSTS   ADDRESS                                                    PORTS   AGE
ingress-demo1   alb     *       cwvj-ingress-demo1-351634829.us-east-2.elb.amazonaws.com   80      33s
```

> **Allow up to  ~3 minutes** for the external ALB to be provisioned after applying the Ingress manifest. You can monitor its creation in the AWS Console under EC2 → Load Balancers.

---


### **Step 5: End-to-End Verification**

Before testing, let’s connect the dots by walking through the end-to-end routing flow for each path: `/iphone`, `/android`, and `/`. This will help solidify how Ingress, Services, Deployments, and AWS Load Balancer Controller work together to route external traffic to internal workloads.

---

#### **Request Flow: How Traffic Reaches the Pods (with Target Type = `ip`)**

* When an external user accesses a URL like `/iphone`, the DNS name resolves to an **AWS Application Load Balancer (ALB)** that was provisioned by the AWS Load Balancer Controller based on the Ingress resource.
* The ALB evaluates listener rules generated from the Ingress manifest. Based on the path (`/iphone`, `/android`, or `/`), it forwards the request to a specific **Target Group**.
* Each Target Group is associated with a Kubernetes Service, which acts as a logical bridge between the Ingress rule and the backend Pods. The Service’s **label selector** is crucial—it tells the Ingress Controller which Pods should be registered in the Target Group. While live traffic bypasses the Service and flows directly to Pod IPs (since target-type is `ip`), the Service remains essential for **target discovery, health check path annotations**, and **routing logic definition** within the Ingress resource.
* Because the `target-type` is set to `ip`, the Target Group contains **Pod IPs** directly. Requests are forwarded from the ALB straight to the container port `5678` on each Pod, bypassing NodePorts and reducing unnecessary network hops.

In summary:

```
User → AWS ALB → Target Group (based on Kubernetes Service) → Pod
```
---

### **Test the Routing**

Once the ALB is fully provisioned and all services report healthy targets, test routing via the ALB's DNS name.

#### **iPhone Users Route**

```
http://cwvj-ingress-demo1-351634829.us-east-2.elb.amazonaws.com/iphone
```

Expected Output:

```
iPhone Users
Welcome to Cloud With VarJosh
```

This is served by the `iphone-deploy` pods, with static HTML under `/iphone/index.html`.

---

#### **Android Users Route**

```
http://cwvj-ingress-demo1-351634829.us-east-2.elb.amazonaws.com/android
```

Expected Output:

```
Android Users
Welcome to Cloud With VarJosh
```

This is served by the `android-deploy` pods from `/android/index.html`.

---

#### **Desktop Users Route (Default/Catch-All)**

```
http://cwvj-ingress-demo1-351634829.us-east-2.elb.amazonaws.com/
```

Expected Output:

```
Desktop Users
Welcome to Cloud With VarJosh
```

This route is matched last and acts as the catch-all fallback. If the request path doesn't match `/iphone` or `/android`, it routes to `desktop-svc`, which serves from the root (`/index.html`).

---

> The catch-all route (`/`) is not a true "default backend" as per Kubernetes Ingress spec, but effectively behaves like one by being the last rule in the list. This is compatible with AWS Load Balancer Controller, which does not support `defaultBackend`.

---

### **Verify the Ingress Configuration**

You can use the following command to inspect your Ingress setup and ensure everything is registered as expected:

```bash
kubectl describe ingress ingress-demo1
```

Look for:

* Ingress class (`alb`)
* Load balancer hostname (ALB DNS name)
* Path rules and associated backends
* Events showing successful sync with the AWS Load Balancer Controller

---

### **Verify from the AWS Console**

To cross-check that your Ingress configuration has been reflected properly:

Navigate to **EC2 → Load Balancers** and locate the ALB with the name defined in:

```yaml
alb.ingress.kubernetes.io/load-balancer-name: cwvj-ingress-demo1
```

Then verify:

* **Scheme** is set to `internet-facing`
* **Listener** is configured on port `80`
* **Target Groups** are created for `iphone-svc`, `android-svc`, and `desktop-svc` with target type `ip`
* **Health Check** settings for each target group match the annotations set on the respective Service
* **Routing Rules** map `/iphone`, `/android`, and `/` paths to the correct target groups


By completing this verification, you confirm that your AWS ALB was dynamically provisioned and fully configured based on the Kubernetes-native Ingress resource—bridging the gap between Kubernetes and AWS networking seamlessly.

---


### **Step 6: Cleanup**

#### **Cleanup**

To remove the Kubernetes resources created during this demo, you can run:

```bash
kubectl delete -f .
```

> This command assumes all manifest files (`Namespace`, `Deployments`, `Services`, `Ingress`) are present in the current directory and were originally applied from here.

If you prefer to delete resources manually, use:

```bash
kubectl delete ingress ingress-demo1
kubectl delete svc iphone-svc android-svc desktop-svc
kubectl delete deploy iphone-deploy android-deploy desktop-deploy
kubectl delete ns app1-ns
```

This will:

* Remove the **Ingress** and associated **AWS Application Load Balancer**
* Delete **target groups**, **listeners**, and **security groups** that were provisioned dynamically

> **Note**: If you're done with all demos and want to completely clean up your EKS environment, you can delete the entire cluster using:

```bash
eksctl delete cluster --name cwvj-ingress-demo
```

> However, since we will build upon the same cluster in **Demo 2**, do **not delete the cluster** yet.

---


### Conclusion

In this lecture, we laid a strong foundation for understanding Kubernetes Ingress — both conceptually and in real-world scenarios.

- You learned **why Ingress is preferred** over NodePort or LoadBalancer services in production.
- We demystified **what an Ingress resource actually does**, and how it works with an **Ingress Controller** to configure an external HTTP(S) Load Balancer.
- You saw a **real-world walkthrough** of how traffic flows from users to backend pods.
- And finally, we wrapped up with **Demo 1**, where we implemented **Path-Based Routing** on a **Multi-AZ Amazon EKS cluster** using the **AWS ALB Ingress Controller**.

In the next lecture (Day 50), we’ll continue with more advanced setups including **TLS with ACM, Route 53 for domain mapping, and name-based routing using subdomains.**

---

### References

* Kubernetes Ingress Docs:
  [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)

* AWS Load Balancer Controller GitHub:
  [https://github.com/kubernetes-sigs/aws-load-balancer-controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller)

* AWS Load Balancer Controller Documentation:
  [https://kubernetes-sigs.github.io/aws-load-balancer-controller/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)

* Ingress Annotations (AWS-specific):
  [https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/)

* AWS Blog – ALB Ingress Controller for Kubernetes:
  [https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/](https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/)

---



