
# DevOps Implementation on OpenTelemetry E-Commerce Microservices

A full-stack DevOps pipeline implemented on a microservices-based e-commerce platform, originally cloned from OpenTelemetry. This project demonstrates containerization, infrastructure as code, Kubernetes-based orchestration, and CI/CD on AWS.


---


## 🐳 Containerization

Three microservices — **Ad Service (Java)**, **Product Catalog Service (Go)**, and **Recommendation Service (Python)** — were containerized using Docker and pushed to Docker Hub. Each service, located in the `src/` directory, was first tested using its native runtime (`go build`, `./gradlew`, `python`) before writing Dockerfiles.

---

### 🟡 Ad Service (Java)

```bash
cd src/adservice
vim Dockerfile
```

```dockerfile
# Build Stage
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /usr/src/app
COPY gradlew* settings.gradle* build.gradle ./
COPY ./gradle ./gradle
RUN chmod +x ./gradlew && ./gradlew && ./gradlew downloadRepos
COPY . . && COPY ./pb ./proto
RUN ./gradlew installDist -PprotoSourceDir=./proto

# Runtime Stage
FROM eclipse-temurin:21-jre
WORKDIR /usr/src/app
COPY --from=builder /usr/src/app ./
ENV AD_PORT=9099
ENTRYPOINT ["./build/install/opentelemetry-demo-ad/bin/Ad"]
```

**Build and run:**

```bash
docker build -t anirudhnegi03/adservice:v1 .
docker run -p 9099:9099 anirudhnegi03/adservice:v1
```

---

### 🟢 Product Catalog Service (Go)

```bash
cd src/productcatalogservice
vim Dockerfile
```

```dockerfile
# Stage 1: Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /usr/src/app
COPY . .
RUN go mod download
RUN go build -o /go/bin/product-catalog ./

# Stage 2: Runtime stage
FROM alpine AS release
WORKDIR /usr/src/app
COPY ./products ./products
COPY --from=builder /go/bin/product-catalog ./
ENV PRODUCT_CATALOG_PORT=8088
ENTRYPOINT ["./product-catalog"]
```

**Build and run:**

```bash
docker build -t anirudhnegi03/product-catalog:v2 .
docker run -p 8088:8088 anirudhnegi03/product-catalog:v2
```

---

### 🔵 Recommendation Service (Python)

```bash
cd src/recommendationservice
vim Dockerfile
```

```dockerfile
FROM python:3.12-slim-bookworm AS base
WORKDIR /usr/src/app
COPY requirements.txt ./
RUN pip install --upgrade pip
RUN pip install -r requirements.txt
COPY . .
ENTRYPOINT ["python", "recommendation_server.py"]
```

**Build and run:**

```bash
docker build -t anirudhnegi03/recommendation-service:v1 .
docker run -p 5001:5001 anirudhnegi03/recommendation-service:v1
```

---

## 📤 Pushing Images to Docker Hub

After successfully building each Docker image locally, the images were tagged and pushed to Docker Hub under the namespace `anirudhnegi03`.

> Make sure you're logged in to Docker Hub first:

```bash
docker login
```

### 🟡 Ad Service

```bash
docker tag anirudhnegi03/adservice:v1 anirudhnegi03/adservice:v1
docker push anirudhnegi03/adservice:v1
```

### 🟢 Product Catalog Service

```bash
docker tag anirudhnegi03/product-catalog:v2 anirudhnegi03/product-catalog:v2
docker push anirudhnegi03/product-catalog:v2
```

### 🔵 Recommendation Service

```bash
docker tag anirudhnegi03/recommendation-service:v1 anirudhnegi03/recommendation-service:v1
docker push anirudhnegi03/recommendation-service:v1
```

> ✅ These pushed images were later referenced in Kubernetes manifests for cloud deployment.


---



## ⚙️ Infrastructure as Code (IaC) with Terraform

To provision secure, scalable, and reproducible AWS infrastructure for this project, **Terraform** was used along with **AWS CLI** and IAM credentials.

---

### 🛠️ Setup and Authentication

1. **Create an IAM user** in AWS Console and generate:
   - **Access Key ID**
   - **Secret Access Key**

2. **Install AWS CLI**  
   Follow the official [AWS CLI installation guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

3. **Configure AWS credentials**:

```bash
aws --version         # Verify AWS CLI is installed
aws configure         # Enter access key, secret key, region, output format
```

This ensures Terraform can authenticate and interact with your AWS account.

---

### 🗂 Directory Structure

All infrastructure code resides in the `eks/` directory:

```
eks/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfstate (generated after apply)
├── backend/
│   ├── outputs.tf
│   └── main.tf
└── modules/
    ├── vpc/
    └── eks/
```

---

### ☁️ Remote Backend Setup (S3 + DynamoDB)

Terraform state is stored remotely to enable collaboration and prevent state corruption.  
You must create:

- An **S3 bucket** to store the `.tfstate` file
- A **DynamoDB table** to manage state locking

Example Terraform backend setup:

```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "demo-terraform-eks-state-s3-bucket-ani"

  lifecycle {
    prevent_destroy = false
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-eks-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

---

### 🧾 `main.tf` Configuration

This is the primary Terraform configuration that sets up the backend, provider, VPC, and EKS cluster.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "demo-terraform-eks-state-s3-bucket-ani"
    key            = "terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-eks-state-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.region
}

module "vpc" {
  source = "./modules/vpc"

  vpc_cidr             = var.vpc_cidr
  availability_zones   = var.availability_zones
  private_subnet_cidrs = var.private_subnet_cidrs
  public_subnet_cidrs  = var.public_subnet_cidrs
  cluster_name         = var.cluster_name
}

module "eks" {
  source = "./modules/eks"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
  node_groups     = var.node_groups
}
```

---

### 🔧 Provisioning Steps

```bash
cd eks
terraform init     # Initializes providers and backend
terraform plan     # Preview changes
terraform apply    # Apply the infrastructure plan
```

Terraform will provision:

- Custom VPC with public/private subnets
- EKS cluster
- EC2 worker nodes
- Remote backend in S3 + DynamoDB for state

---

### ✅ What’s Provisioned

| Resource      | Description                                                                 |
|---------------|-----------------------------------------------------------------------------|
| **VPC**       | Custom VPC with public and private subnets                                  |
| **EKS**       | Managed Kubernetes cluster with autoscaling node groups                     |
| **EC2 Nodes** | Worker nodes in private subnets running Kubernetes workloads                |
| **S3 Bucket** | Stores remote state (`terraform.tfstate`)                                   |
| **DynamoDB**  | State locking to prevent concurrent apply from corrupting infrastructure     |

---

### 🧾 Summary

This setup ensures:

- Modular, production-grade infrastructure
- Secure state management with S3 and DynamoDB
- Reusability and automation using Terraform modules
- Compatibility with DevOps and CI/CD practices

> You can verify resources in the AWS Console under EC2, S3, DynamoDB, and EKS dashboards.


---


## 🚀 Deploying Project to Kubernetes (EKS)

This project deploys a collection of containerized microservices to a **Kubernetes-managed EKS cluster** using `kubectl` and YAML manifests.

---

### 🧩 Step 1: Update `kubectl` Config

After provisioning the EKS cluster, you must configure your local `kubectl` to communicate with it:

```bash
aws eks update-kubeconfig --region ap-south-1 --name my-eks-cluster
```

This command updates your `~/.kube/config` file with:
- API endpoint for your EKS cluster
- IAM authentication
- Cluster context

Without this step, `kubectl` will not know about your EKS cluster and cannot communicate with it.

---

### 🔍 Why Is This Necessary?

When you create an EKS cluster (via AWS CLI or Terraform), it sets up:
- The control plane
- IAM roles
- Subnets and networking  
But it does **not** configure your local CLI tools. That’s why `aws configure` and `aws eks update-kubeconfig` are required — so tools like:
- `aws` CLI
- `kubectl`
- `terraform`
can authenticate and communicate with AWS services on your behalf.

---

### 📦 Microservice Deployment Structure

All Kubernetes YAML files are located under the `kubernetes/` directory:

```
kubernetes/
├── adservice/
│   ├── deployment.yaml
│   └── service.yaml
├── productcatalogservice/
│   ├── deployment.yaml
│   └── service.yaml
├── recommendationservice/
│   ├── deployment.yaml
│   └── service.yaml
├── service-account.yaml
└── complete-deploy.yaml
```

- `deployment.yaml`: Defines how many Pods to run and what container to use.
- `service.yaml`: Exposes the Pod (internally or externally).
- `service-account.yaml`: Grants permissions to Pods to interact with AWS services via IAM.

---

### ⚙️ Apply YAML Files to Kubernetes

You can deploy all services at once using:

```bash
kubectl apply -f kubernetes/service-account.yaml
kubectl apply -f kubernetes/complete-deploy.yaml
```

This applies the following:
- All three microservices (Ad, Product Catalog, Recommendation)
- Their deployments and services
- A Kubernetes ServiceAccount (`opentelemetry-demo`) with appropriate labels and metadata

---

### 🛡️ service-account.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: opentelemetry-demo
  labels:
    opentelemetry.io/name: opentelemetry-demo
    app.kubernetes.io/instance: opentelemetry-demo
    app.kubernetes.io/name: opentelemetry-demo
    app.kubernetes.io/version: "1.12.0"
    app.kubernetes.io/part-of: opentelemetry-demo
```

This service account is used by microservices that require identity within the cluster or permissions to interact with AWS services when integrated with IAM Roles for Service Accounts (IRSA).

---

### 🌐 Understanding Kubernetes Service Types

By default, all services in Kubernetes use:

```yaml
type: ClusterIP
```

This means the service is only accessible **within the cluster**. Not even an EC2 or Lambda in the same VPC can access it.

#### 📌 To allow external access, change to:

- `NodePort`: Exposes service on a static port on each node (e.g., `node-ip:30001`).
- `LoadBalancer`: Creates a **cloud-managed load balancer** to expose the service publicly.

---

### 🌍 Exposing Frontend with LoadBalancer

To make the frontend accessible over the internet:

1. Edit the frontend proxy service type:

```bash
kubectl edit svc opentelemetry-demo-frontendproxy
```

2. Change:

```yaml
type: ClusterIP
```

to:

```yaml
type: LoadBalancer
```

3. Save and exit.

4. Visit the AWS Console → EC2 → Load Balancers → copy the **DNS name** of the Load Balancer.

Now, you can access the app publicly via:

```
http://<load-balancer-dns>:<frontend-port>
```

---

### ✅ Summary

| Component       | Description                                         |
|------------------|-----------------------------------------------------|
| `kubectl` config | Set up using `aws eks update-kubeconfig`            |
| Deployments      | Managed using `deployment.yaml` for each service    |
| Services         | Initially `ClusterIP`, updated to `LoadBalancer`    |
| Load Balancer    | Automatically created by AWS when type is changed   |
| Access URL       | Public DNS of the AWS Load Balancer                 |
| Service Account  | Grants Pods IAM-based AWS access securely           |

This approach mirrors real-world enterprise deployments — allowing local development and seamless cloud deployment using EKS and Kubernetes best practices.


---


## 🌐 Implementing Ingress to Expose Frontend Proxy

Previously, we exposed the **frontend-proxy** service using a Kubernetes `Service` of type `LoadBalancer`, which automatically created an AWS ALB. While effective, this approach:

- Is **costly** (creates one ALB per service)
- Lacks **routing flexibility**

To solve this, we implemented **Ingress** to use a **single AWS ALB** with **fine-grained host and path-based routing**.

---

### 🧹 Step 1: Clean Up Previous LoadBalancer

To avoid duplicate ALBs and reduce costs, we **reverted the frontend proxy service** from `LoadBalancer` to `NodePort`:

```bash
kubectl edit svc opentelemetry-demo-frontendproxy
```

Change:

```yaml
type: LoadBalancer
```

to:

```yaml
type: NodePort
```

This automatically deletes the previous ALB provisioned for the frontend service.

---

### 📝 Step 2: Create `ingress.yaml`

Created under `kubernetes/frontendproxy/ingress.yaml`, this defines how Ingress will expose the service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-proxy
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: example.com
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
```

---

### 📌 Explanation of Fields

| Field                                  | Purpose                                                       |
|----------------------------------------|---------------------------------------------------------------|
| `ingressClassName: alb`                | Ensures ALB Ingress Controller handles this resource          |
| `alb.ingress.kubernetes.io/scheme`     | Sets ALB as **internet-facing**                               |
| `alb.ingress.kubernetes.io/target-type`| Routes traffic to **Pod IPs** instead of Node IPs             |
| `host: example.com`                    | Ingress will only respond to requests with this host header   |
| `path: "/"` with `pathType: Prefix`    | Forwards all traffic from `/` to the specified backend service|

---

### 🚀 Step 3: Apply the Ingress

```bash
kubectl apply -f kubernetes/frontendproxy/ingress.yaml
```

This triggers the **ALB Ingress Controller** to automatically provision a new **Application Load Balancer** in AWS.

You can verify it using:

```bash
kubectl get ingress
```

or via AWS Console → EC2 → Load Balancers.

---

### 🧪 Step 4: Test Host-Based Routing

Since we configured the Ingress for `example.com`, the ALB **only accepts traffic** with that host in the header.

#### ❌ Direct IP/DNS Access

Accessing the ALB DNS name or IP directly **will not work**:

```
http://<ALB-DNS>    ❌ (403 or 404)
```

#### ✅ Simulate Domain with `/etc/hosts` (Local DNS Override)

To fake `example.com`:

1. Get the ALB’s DNS or IP
2. Edit your local `/etc/hosts` file:

```bash
sudo vim /etc/hosts
```

Add this line:

```
<ALB-IP> example.com
```

Now your system resolves `example.com` to the ALB.

3. Visit:

```
http://example.com
```

It should work now!

> Note: If it doesn’t work in Firefox due to DNS caching, try Chrome or another browser.

---

### ✅ Summary

| Step                         | Result                                                       |
|------------------------------|--------------------------------------------------------------|
| Reverted service to NodePort | Deleted old expensive ALB                                    |
| Created Ingress YAML         | Defined host/path rules for frontend                         |
| Applied Ingress              | Triggered automatic ALB provisioning by Ingress Controller   |
| Edited `/etc/hosts`          | Simulated real domain to test Ingress rules                  |
| Verified                     | `example.com` correctly routed traffic to frontend service   |

---

## 🚀 Continuous Integration for Product Catalog Microservice

This section documents the CI setup for the **Product Catalog** microservice using **GitHub Actions**.


### 🧩 GitHub Actions Workflow Structure

Each GitHub Actions workflow (`.github/workflows/product-catalog-ci.yaml`) contains:

- **Name** – Identifier for the workflow  
- **Trigger** – When the workflow should run (e.g., on PR to `main`)
- **Jobs** – Logical units like `build`, `test`, `lint`, `docker`, etc.

---

### ⚙️ GitHub Actions Workflow: CI

```yaml
name: product-catalog-ci

on:
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Go 1.22
        uses: actions/setup-go@v2
        with:
          go-version: '1.22'

      - name: Build
        run: |
          cd src/product-catalog
          go mod download
          go build -o product-catalog-service main.go
      
      - name: Unit tests
        run: |
          cd src/product-catalog
          go test ./...

  code-quality:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v3
        with:
          version: latest
          args: run ./src/product-catalog/...

  docker:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Docker
        uses: docker/setup-buildx-action@v1

      - name: Login to Docker
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Docker Build & Push
        uses: docker/build-push-action@v6
        with:
          context: src/product-catalog
          file: src/product-catalog/Dockerfile
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/product-catalog:${{ github.run_id }}

  updateK8s:
    runs-on: ubuntu-latest
    needs: [build, docker, code-quality]
    steps:
      - name: Checkout code with write access
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Update image tag in Kubernetes deployment manifest
        run: |
          sed -i 's|image: .*|image: '"${{ secrets.DOCKER_USERNAME }}"'/product-catalog:'"${{ github.run_id }}"'|' kubernetes/productcatalog/deploy.yaml

      - name: Commit and push changes
        run: |
          git config --global user.name "Anirudh Negi"
          git config --global user.email "4nirudhnegi@gmail.com"
          git add kubernetes/productcatalog/deploy.yaml
          git commit -m "[CI]: Update product catalog image tag"
          git push origin HEAD:main -f
```
## ✅ CI Testing Workflow (Step-by-Step)

```bash
# Create a test branch
git checkout -b test-ci-5

# (Added some spaces at the end of main.go to trigger CI)

# Check changes
git status

# Stage and commit
git add .
git commit -m "Test CI for product catalog"

# Push to GitHub
git push origin test-ci-5

<img width="874" height="640" alt="Screenshot 2025-07-14 220510" src="https://github.com/user-attachments/assets/f25fefa5-f071-4205-9c9f-c534fc0a865d" />
```
## 🚢 CD with Argo CD and GitOps

### 🔸 What is CD?

**Continuous Delivery (CD)** ensures that once a new Docker image is built and pushed (via CI), the corresponding Kubernetes manifests are automatically applied to the cluster — completing the full CI/CD lifecycle.

We achieve this with **GitOps**, where a tool like **Argo CD** watches the Git repo and syncs Kubernetes changes to the cluster automatically.

---

### 🧠 Why GitOps with Argo CD?

| Feature                 | Description                                                                 |
|------------------------|-----------------------------------------------------------------------------|
| 🔄 **Auto Sync**        | Argo CD automatically syncs Git changes (e.g., new image tags) to the cluster |
| 🔒 **Source of Truth**  | Git repo is the single source of truth. Manual changes in cluster are reverted |
| ⚠️ **Drift Detection** | If a resource is changed manually, Argo CD detects and corrects it             |
| 🧭 **Central Control**  | Argo CD can manage multiple clusters from one dashboard                        |

---

### ⚙️ Argo CD Installation Steps

```bash
# Step 1: Create Argo CD namespace
kubectl create namespace argocd

# Step 2: Install Argo CD using official manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 🌐 Expose Argo CD UI

```bash
# Change service type from ClusterIP to LoadBalancer
kubectl edit svc argocd-server -n argocd
```
After ~2 minutes, you'll get an external IP to access the Argo CD UI.

---

### 🔑 Access Argo CD

```bash
# Get the admin password (base64 decode required)
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
```
Use:
- **Username**: `admin`
- **Password**: base64-decoded value from `argocd-initial-admin-secret`

---

### 🚀 Argo CD Application Setup

In the Argo CD UI:

1. Click on **Create Application**
2. Set **Name**: `product-catalog-service`
3. Set **Project**: `default`
4. Choose **Sync Policy**: `Automatic`
5. Set **Repository URL**: your GitHub repository URL
6. Set **Revision**: `HEAD`
7. Set **Path**: `kubernetes/productcatalog/`
8. Set **Cluster URL**: `https://kubernetes.default.svc`
9. Set **Namespace**: `default`

Click **Create**. Argo CD will:

- Clone the repo
- Detect the new Docker tag in `deploy.yaml`
- Deploy a new ReplicaSet with the updated image

---
<img width="899" height="453" alt="Screenshot 2025-07-14 222627" src="https://github.com/user-attachments/assets/0b5141cd-2b38-4554-8a59-be7330d948c2" />


### 🔎 Validation & Pod Status

```bash
# Check the running ReplicaSet
kubectl get rs

# Edit to verify the new image tag (optional)
kubectl edit rs <replica-set-name>
```
<img width="893" height="311" alt="image" src="https://github.com/user-attachments/assets/29142a9b-277c-4cc2-865a-c7f6ec44345c" />

<img width="898" height="501" alt="image" src="https://github.com/user-attachments/assets/c767dd20-f806-4792-813b-094a0b0bdfe0" />


✅ A new image `anirudhnegi03/product-catalog:<run_id>` should be reflected in the deployment.

⚠️ If the Pod is stuck in `Pending`, it's likely due to resource limits in the cluster (e.g., not enough CPU or memory).
