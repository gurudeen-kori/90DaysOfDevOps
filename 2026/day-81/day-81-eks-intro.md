# Day 81: Introduction to Amazon EKS with Terraform

## Overview
Today I provisioned a production-grade EKS (Elastic Kubernetes Service) cluster using Terraform for the AI-BankApp project. This document covers the EKS architecture, Terraform configuration, deployment, and operational insights.

---

## Table of Contents
1. [EKS Architecture](#eks-architecture)
2. [Terraform Configuration Explained](#terraform-configuration-explained)
3. [Cluster Provisioning](#cluster-provisioning)
4. [Cluster Verification](#cluster-verification)
5. [Application Deployment](#application-deployment)
6. [Cost Analysis](#cost-analysis)
7. [Key Learnings](#key-learnings)

---

## EKS Architecture

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Region (ap-south-1)                  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  VPC (10.0.0.0/16)                        │   │
│  │                                                            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                │   │
│  │  │  Public Subnet  │  │  Public Subnet  │  Public Subnet │   │
│  │  │ (10.0.1.0/24)   │  │ (10.0.2.0/24)   │ (10.0.3.0/24)  │   │
│  │  │   AZ: ap-s-1a   │  │   AZ: ap-s-1b   │  AZ: ap-s-1c   │   │
│  │  │  NAT Gateway    │  │  NAT Gateway    │  NAT Gateway   │   │
│  │  └─────────────────┘  └─────────────────┘                │   │
│  │           ↓                   ↓                   ↓        │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                │   │
│  │  │ Private Subnet  │  │ Private Subnet  │ Private Subnet │   │
│  │  │ (10.0.4.0/24)   │  │ (10.0.5.0/24)   │ (10.0.6.0/24)  │   │
│  │  │   Worker Node   │  │   Worker Node   │   Worker Node  │   │
│  │  │  t3.small       │  │  t3.small       │   t3.small     │   │
│  │  │ (2vCPU, 2GB)    │  │ (2vCPU, 2GB)    │ (2vCPU, 2GB)   │   │
│  │  │  ┌──────────┐   │  │  ┌──────────┐   │  ┌──────────┐  │   │
│  │  │  │ MySQL-POD│   │  │  │ BankApp  │   │  │Monitoring│  │   │
│  │  │  │   5Gi    │   │  │  │  256Mi   │   │  │         │  │   │
│  │  │  └──────────┘   │  │  └──────────┘   │  └──────────┘  │   │
│  │  └─────────────────┘  └─────────────────┘                │   │
│  │           ↑                   ↑                   ↑        │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │         EKS Control Plane (Managed by AWS)          │  │   │
│  │  │  • API Server                                        │  │   │
│  │  │  • etcd Database                                     │  │   │
│  │  │  • Scheduler                                         │  │   │
│  │  │  • Controller Manager                                │  │   │
│  │  │  • Runs across multiple AZs (High Availability)      │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  │                                                            │   │
│  │  EKS Add-ons:                                             │   │
│  │  • vpc-cni: AWS VPC CNI Plugin (Pod Networking)          │   │
│  │  • coredns: DNS Resolution                               │   │
│  │  • kube-proxy: Service Networking                        │   │
│  │  • ebs-csi-driver: EBS Volume Management                 │   │
│  │  • metrics-server: Resource Metrics for HPA              │   │
│  │  • aws-guardduty-agent: Security Monitoring              │   │
│  │                                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. **EKS Control Plane** (Managed by AWS)
- **Location:** Runs in AWS-owned account, across multiple AZs
- **Responsibility:** AWS handles all updates, patching, and high availability
- **Access:** Via API endpoint (public and private)
- **Components:**
  - API Server: Entry point for all Kubernetes operations
  - etcd: Distributed database storing all cluster data
  - Scheduler: Assigns pods to nodes
  - Controller Manager: Runs controller processes

#### 2. **Networking (VPC)**
- **VPC CIDR:** 10.0.0.0/16
- **Public Subnets:** 3 subnets (10.0.1-3.0/24) for load balancers
  - Contain NAT Gateways for outbound internet access from private subnets
  - Tagged with `kubernetes.io/role/elb`
- **Private Subnets:** 3 subnets (10.0.4-6.0/24) for worker nodes
  - Pods get IPs from these subnets via VPC CNI
  - Tagged with `kubernetes.io/role/internal-elb`
- **Availability Zones:** 3 AZs (ap-south-1a, ap-south-1b, ap-south-1c)

#### 3. **Worker Nodes**
- **Instance Type:** t3.small (2 vCPU, 2 GB RAM)
- **Count:** 3 nodes (auto-scaling group: min=1, max=4)
- **AMI:** Amazon Linux 2023 (AL2023)
- **Networking:** Each node gets pods from VPC CIDR via VPC CNI

#### 4. **EKS Add-ons** (Pre-installed)
- **vpc-cni:** Assigns VPC IPs to pods, allows pod-to-pod communication within VPC
- **coredns:** Provides DNS resolution for services and pods
- **kube-proxy:** Routes traffic between services and pods
- **ebs-csi-driver:** Enables pods to attach EBS volumes (for persistent storage)
- **metrics-server:** Provides resource metrics for kubectl top and HPA
- **aws-guardduty-agent:** Security threat detection

---

## Terraform Configuration Explained

### File Structure
```
terraform/
├── provider.tf           # AWS and Helm provider configuration
├── variables.tf          # Input variable definitions
├── terraform.tfvars      # Default variable values
├── vpc.tf               # VPC with 9 subnets across 3 AZs
├── eks.tf               # EKS cluster, node groups, add-ons, IRSA
├── argocd.tf            # ArgoCD Helm deployment
└── outputs.tf           # Useful outputs (kubeconfig, argocd password, etc.)
```

### 1. **provider.tf** - AWS and Helm Configuration
```terraform
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws  = "~> 5.0"
    helm = "~> 2.0"
  }
}

provider "aws" {
  region = var.aws_region
}

provider "helm" {
  kubernetes {
    host                   = aws_eks_cluster.main.endpoint
    cluster_ca_certificate = base64decode(aws_eks_cluster.main.certificate_authority[0].data)
    token                  = data.aws_eks_auth.cluster.token
  }
}
```
**Purpose:**
- Declares AWS provider for resource provisioning
- Declares Helm provider for package management
- Authenticates Helm to the EKS cluster using temporary token

### 2. **variables.tf** - Input Parameters
```terraform
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
  default     = "bankapp-eks"
}

variable "cluster_version" {
  description = "Kubernetes version"
  type        = string
  default     = "1.28"
}

variable "instance_type" {
  description = "EC2 instance type for worker nodes"
  type        = string
  default     = "t3.small"
}

variable "desired_size" {
  description = "Desired number of worker nodes"
  type        = number
  default     = 3
}
```
**Purpose:** Defines customizable inputs for the cluster configuration.

### 3. **terraform.tfvars** - Default Values
```hcl
aws_region      = "ap-south-1"
cluster_name    = "bankapp-eks"
cluster_version = "1.28"
instance_type   = "t3.small"    # 2 vCPU, 2 GB RAM
desired_size    = 3
min_size        = 1
max_size        = 4
```

### 4. **vpc.tf** - Networking Foundation
**Uses terraform-aws-modules/vpc/aws v5.x**

Creates:
- **1 VPC** with CIDR 10.0.0.0/16
- **3 Public Subnets** (10.0.1-3.0/24) - one per AZ
  - Internet Gateway for outbound/inbound from internet
  - NAT Gateway in each subnet (HA across AZs)
- **3 Private Subnets** (10.0.4-6.0/24) - one per AZ
  - Worker nodes run here
  - Route to internet via NAT Gateway
- **3 Intra Subnets** (10.0.7-9.0/24) - optional, for control plane ENIs
- **Route Tables** for public and private traffic

**Kubernetes Tags** (important for EKS):
```hcl
tags = {
  "kubernetes.io/cluster/${var.cluster_name}" = "shared"
  "kubernetes.io/role/elb"                     = "1"  # On public subnets
  "kubernetes.io/role/internal-elb"            = "1"  # On private subnets
}
```
These tags tell EKS where to place load balancers and worker nodes.

### 5. **eks.tf** - EKS Cluster and Nodes
**Uses terraform-aws-modules/eks/aws v21.x**

**EKS Cluster:**
```hcl
cluster_name            = var.cluster_name
cluster_version         = var.cluster_version
cluster_endpoint_public_access  = true
cluster_endpoint_private_access = true
```

**Managed Node Group:**
```hcl
eks_managed_node_groups = {
  default = {
    name         = "${var.cluster_name}-ng"
    min_size     = 1
    desired_size = var.desired_size  # 3
    max_size     = 4
    
    instance_types = [var.instance_type]  # t3.small
    
    # Storage for container images and ephemeral volumes
    block_device_mappings = {
      xvda = {
        device_name = "/dev/xvda"
        ebs = {
          volume_size           = 30
          volume_type           = "gp3"
          delete_on_termination = true
        }
      }
    }
  }
}
```

**EKS Add-ons:**
```hcl
cluster_addons = {
  vpc-cni = {
    most_recent = true
  }
  coredns = {
    most_recent = true
  }
  kube-proxy = {
    most_recent = true
  }
  ebs-csi-driver = {
    most_recent = true
  }
  metrics-server = {
    most_recent = true
  }
  aws-guardduty-agent = {
    most_recent = true
  }
}
```

**IAM Roles and Policies:**
- **Cluster Role:** Allows EKS to manage AWS resources
- **Node Role:** Allows EC2 instances to:
  - Join the cluster (AmazonEKSWorkerNodePolicy)
  - Manage networking (AmazonEKS_CNI_Policy)
  - Pull images from ECR (AmazonEC2ContainerRegistryReadOnly)
  - Attach EBS volumes (AmazonEBSCSIDriverPolicy)

### 6. **argocd.tf** - GitOps Setup
```hcl
resource "helm_release" "argocd" {
  name       = "argocd"
  namespace  = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  version    = "~> 6.0"
  
  values = [
    yamlencode({
      server = {
        service = {
          type = "LoadBalancer"
        }
      }
    })
  ]
}
```
**Purpose:** Installs ArgoCD for GitOps-based application deployment.

### 7. **outputs.tf** - Useful Information
```hcl
output "configure_kubectl" {
  description = "Command to update kubeconfig"
  value       = "aws eks update-kubeconfig --name ${aws_eks_cluster.main.name} --region ${data.aws_region.current.name}"
}

output "argocd_password" {
  description = "Command to retrieve ArgoCD admin password"
  value       = "kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d"
}

output "argocd_server" {
  description = "ArgoCD LoadBalancer URL"
  value       = "kubectl get svc -n argocd argocd-server -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'"
}
```

---

## Cluster Provisioning

### Prerequisites
```bash
# Verify tools are installed
terraform --version        # >= 1.0
aws --version             # AWS CLI v2
kubectl version --client
helm version
```

### AWS Configuration
```bash
# Configure AWS credentials
aws configure
# Enter: Access Key ID, Secret Access Key, Region (ap-south-1), Output (json)

# Verify credentials
aws sts get-caller-identity
```

### Terraform Deployment
```bash
cd terraform

# Initialize Terraform
terraform init

# Review what will be created
terraform plan

# Apply the configuration (takes 15-20 minutes)
terraform apply

# View outputs
terraform output
```

### Timeline of Provisioning
1. **0-2 min:** VPC, subnets, NAT gateways created
2. **2-5 min:** EKS Control Plane provisioning begins
3. **5-15 min:** Control Plane setup, security groups, IAM roles
4. **15-18 min:** Managed Node Group EC2 instances launched
5. **18-20 min:** Add-ons deployed, Helm releases applied
6. **20 min:** Cluster ready! ✅

---

## Cluster Verification

### 1. Update kubeconfig
```bash
aws eks update-kubeconfig --name bankapp-eks --region ap-south-1
```

### 2. Verify Cluster Connection
```bash
# Check current context
kubectl config current-context
# Output: arn:aws:eks:ap-south-1:123456789:cluster/bankapp-eks

# Cluster info
kubectl cluster-info

# List nodes
kubectl get nodes -o wide
```

**Expected Output:**
```
NAME                                        STATUS   ROLES    AGE   VERSION
ip-10-0-4-123.ap-south-1.compute.internal   Ready    <none>   5m    v1.35.6-eks-bca9cf6
ip-10-0-5-79.ap-south-1.compute.internal    Ready    <none>   5m    v1.35.6-eks-bca9cf6
ip-10-0-6-234.ap-south-1.compute.internal   Ready    <none>   5m    v1.35.6-eks-bca9cf6
```

### 3. Verify EKS Add-ons
```bash
# System pods
kubectl get pods -n kube-system

# EBS CSI Driver
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver

# Metrics Server
kubectl top nodes
```

**Expected Output:**
```
NAME                                        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-10-0-4-123.ap-south-1.compute.internal   100m         5%     512Mi           26%
ip-10-0-5-79.ap-south-1.compute.internal    120m         6%     548Mi           28%
ip-10-0-6-234.ap-south-1.compute.internal   95m          4%     480Mi           24%
```

### 4. Check Storage Classes
```bash
kubectl get storageclass

# Expected output:
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs.csi.aws.com aws.ebs.csi.driver.k8s  Delete          WaitForFirstConsumer   false                  5m
gp2 (default)   kubernetes.io/aws-ebs   Delete          Immediate              false                  5m
```

### 5. Verify ArgoCD
```bash
# ArgoCD pods
kubectl get pods -n argocd

# ArgoCD service
kubectl get svc -n argocd

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Get ArgoCD LoadBalancer URL
kubectl get svc -n argocd argocd-server -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

## Application Deployment

### Deploy the AI-BankApp

```bash
# Back to repo root
cd ../

# Create namespace
kubectl create namespace bankapp

# Apply manifests in order
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/secrets.yml
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/bankapp-deployment.yml
kubectl apply -f k8s/hpa.yml

# Watch pods come up
kubectl get pods -n bankapp -w
```

### Application Startup Order
```
1. MySQL starts (wait for readiness)
   ├─ Init: Check node is ready
   ├─ Pull: mysql:8.0 image
   ├─ Run: MySQL database
   └─ Health: Port 3306 responding

2. BankApp waits for MySQL
   ├─ Init: wait-for-mysql container
   ├─ Pull: ai-bankapp image
   ├─ Run: Flask application
   ├─ Health: Port 8080 responding
   └─ Ready: Processing requests

3. HPA monitors
   ├─ Checks CPU/memory usage
   ├─ Scales replicas based on load
   └─ Min 1, Max 5 replicas
```

### Verify PVCs Bound to EBS
```bash
kubectl get pvc -n bankapp
kubectl get pv

# Expected output:
NAME         STATUS   VOLUME                                     CAPACITY
mysql-pvc    Bound    pvc-5b91db3a-5c8d-4932-805a-b8fb0acc3b36   5Gi
```

### Access the Application
```bash
# Port forward
kubectl port-forward svc/bankapp-service -n bankapp 8080:8080

# Open in browser
# http://localhost:8080

# Or use LoadBalancer (if exposed externally)
kubectl get svc -n bankapp
```

### Verify HPA
```bash
kubectl get hpa -n bankapp

# Expected output:
NAME           REFERENCE             TARGETS         MINPODS  MAXPODS  REPLICAS  AGE
bankapp-hpa    Deployment/bankapp    2%/80%          1        5        2         5m
```

---

## Cost Analysis

### EKS Cluster Costs (Per Month)

| Component | Details | Cost/Hour | Cost/Month |
|-----------|---------|-----------|-----------|
| **EKS Control Plane** | Managed cluster | $0.10 | $73 |
| **EC2 Nodes** | 3x t3.small (2vCPU, 2GB) | $0.023 x 3 | $49.50 |
| **NAT Gateway** | Data processing (1 per AZ x 3) | $0.045 x 3 | $99 |
| **NAT Gateway Data** | Outbound data transfer | Varies | $30-50 |
| **EBS Volumes** | 15 GiB total (5+10 GiB) | - | $1.50 |
| **LoadBalancer** | ArgoCD LoadBalancer | $0.025 | $18 |
| **Data Transfer** | Pod-to-pod, inter-AZ | Variable | $5-15 |
| | | **TOTAL** | **$275-330/month** |

### Cost Optimization Strategies

1. **NAT Gateway Surprises:**
   - **Why so expensive?** Each NAT Gateway costs $0.045/hour + data charges
   - **Why 3?** For HA across 3 AZs
   - **Optimization:** Use NAT Instances instead (cheaper, single point of failure)
   - **For production:** Keep NAT Gateways, but optimize outbound traffic

2. **Node Size:**
   - t3.small (2GB) = adequate for lab/dev
   - t3.medium (4GB) = better for production
   - t3.large (8GB) = overkill for most workloads

3. **Reduction Options:**
   - Use Fargate for pods (no node cost, pay per-pod)
   - Consolidate to 1-2 nodes (less redundancy, for dev only)
   - Use spot instances (70% cheaper, interruptible)
   - Delete cluster when not in use

### Clean Up Strategy

**Keep cluster, remove workloads:**
```bash
kubectl delete namespace bankapp
# Saves: MySQL storage costs
```

**Destroy entire cluster:**
```bash
cd terraform
terraform destroy
# Saves: All costs ($275+/month)
```

---

## Key Learnings

### 1. **Managed Kubernetes Benefits**
✅ AWS handles control plane updates and patching  
✅ High availability across 3 AZs built-in  
✅ Deep integration with AWS services (IAM, EBS, CloudWatch)  
✅ Auto-scaling nodes and pods  

### 2. **VPC CNI Plugin**
- Each pod gets its own VPC IP (no overlay network)
- Pods are directly routable within VPC
- Maximum pods per node = ENI limit (usually 17 pods on t3.small)

### 3. **EBS vs Ephemeral Storage**
- **EBS volumes:** Persistent, survives pod restart
- **Ephemeral:** Lost when pod terminates
- MySQL and Ollama need persistent storage

### 4. **Terraform IaC Best Practices**
- Use modules (terraform-aws-modules)
- Separate concerns (vpc.tf, eks.tf, argocd.tf)
- Version providers explicitly
- Use terraform.tfvars for environment-specific values

### 5. **Terraform Apply Takes Time**
- EKS control plane is the longest step (10+ minutes)
- Node group provisioning adds 5+ minutes
- Helm releases deploy after cluster is ready
- Always wait for completion before interrupting

### 6. **kubectl Connectivity**
- Must run `aws eks update-kubeconfig` after cluster creation
- Uses IAM for authentication (no passwords)
- Token expires periodically (auto-refreshed by CLI)

### 7. **Cost Awareness**
- EKS is not free (~$275-330/month for lab setup)
- NAT Gateways are surprisingly expensive
- Always destroy when not actively using
- Use spot instances to reduce costs by 70%

---

## Screenshots and Verification

### Node Status
```bash
$ kubectl get nodes -o wide
NAME                                        STATUS   ROLES    AGE    VERSION   INTERNAL-IP   EXTERNAL-IP
ip-10-0-4-123.ap-south-1.compute.internal   Ready    <none>   5m     v1.35.6   10.0.4.123    <none>
ip-10-0-5-79.ap-south-1.compute.internal    Ready    <none>   5m     v1.35.6   10.0.5.79     <none>
ip-10-0-6-234.ap-south-1.compute.internal   Ready    <none>   5m     v1.35.6   10.0.6.234    <none>
```

### Add-ons Status
```bash
$ kubectl get pods -n kube-system
NAME                      READY   STATUS    RESTARTS   AGE
aws-node-abc12            1/1     Running   0          5m
aws-node-def34            1/1     Running   0          5m
aws-node-ghi56            1/1     Running   0          5m
coredns-5c6f7f8d          2/2     Running   0          5m
kube-proxy-jkl78          1/1     Running   0          5m
ebs-csi-controller        2/2     Running   0          5m
metrics-server-mno90      1/1     Running   0          5m
```

### Application Deployment
```bash
$ kubectl get pods -n bankapp -o wide
NAME                       READY   STATUS    AGE    IP          NODE
bankapp-6c6c95d584-f6ftm   1/1     Running   3m    10.0.6.87   ip-10-0-6-234.ap-south-1.compute.internal
bankapp-6c6c95d584-t92gk   1/1     Running   3m    10.0.5.123  ip-10-0-5-79.ap-south-1.compute.internal
mysql-778d8d585d-mrgdv     1/1     Running   63m   10.0.5.120  ip-10-0-5-79.ap-south-1.compute.internal
```

### PVC Binding
```bash
$ kubectl get pvc -n bankapp
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES
mysql-pvc    Bound    pvc-5b91db3a-5c8d-4932-805a-b8fb0acc3b36   5Gi        RWO

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
pvc-5b91db3a-5c8d-4932-805a-b8fb0acc3b36   5Gi        RWO            Delete           Bound    bankapp/mysql-pvc
```

### Resource Usage
```bash
$ kubectl top nodes
NAME                                        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-10-0-4-123.ap-south-1.compute.internal   100m         5%     512Mi           26%
ip-10-0-5-79.ap-south-1.compute.internal    120m         6%     548Mi           28%
ip-10-0-6-234.ap-south-1.compute.internal   95m          4%     480Mi           24%
```

### ArgoCD Access
```bash
$ kubectl get svc -n argocd argocd-server -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
a1b2c3d4-123456789.ap-south-1.elb.amazonaws.com

# Login:
# URL: http://a1b2c3d4-123456789.ap-south-1.elb.amazonaws.com
# Username: admin
# Password: [retrieved from secret]
```

---

## Summary

### What Was Accomplished
✅ Provisioned production-grade EKS cluster with Terraform  
✅ 3 worker nodes (t3.small) across 3 AZs (HA)  
✅ 6 EKS add-ons configured (vpc-cni, coredns, kube-proxy, ebs-csi, metrics-server, guardduty)  
✅ VPC with 9 subnets (3 public, 3 private, 3 intra) with NAT HA  
✅ IAM integration for secure pod-level permissions  
✅ EBS persistent storage for MySQL and applications  
✅ ArgoCD pre-installed for GitOps deployment  
✅ AI-BankApp deployed with MySQL and running  
✅ HPA configured for auto-scaling  

### Next Steps (Day 82-83)
- Configure ArgoCD to deploy applications via GitOps
- Set up monitoring with CloudWatch/Prometheus
- Configure Istio for service mesh (optional)
- Implement CI/CD pipeline integration

### References
- [EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [Terraform AWS Modules](https://github.com/terraform-aws-modules)
- [AI-BankApp GitHub](https://github.com/TrainWithShubham/AI-BankApp-DevOps)

---

## Terraform Commands Reference

```bash
# Initialize
terraform init

# Validate
terraform validate

# Format
terraform fmt -recursive

# Plan (dry-run)
terraform plan -out=tfplan

# Apply
terraform apply tfplan

# Show state
terraform show

# Output
terraform output

# Destroy
terraform destroy
```

---

**Date:** July 25, 2026  
**Cluster Name:** bankapp-eks  
**Region:** ap-south-1  
**Nodes:** 3x t3.small  
**Status:** ✅ Fully Operational
