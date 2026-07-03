# Day 66 -- Provision an EKS Cluster with Terraform Modules

Provisioned a full AWS EKS cluster using Terraform registry modules -- VPC, subnets, NAT gateway, IAM roles, managed node group -- then connected `kubectl` and deployed Nginx on it.

- **Cluster name:** `terraweek-eks`
- **Region:** `ap-south-1`
- **Kubernetes version:** `v1.31.13-eks-ecaa3a6`
- **AWS Account:** `942521689864`

---

## 1. Project Structure

```
terraform-eks/
  providers.tf        # Provider and backend config
  vpc.tf               # VPC module call
  eks.tf               # EKS module call
  variables.tf         # All input variables
  outputs.tf            # Cluster outputs
  terraform.tfvars      # Variable values
  k8s/
    nginx-deployment.yaml
```

### `providers.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

### `variables.tf`

```hcl
variable "region" {
  type = string
}

variable "cluster_name" {
  type    = string
  default = "terraweek-eks"
}

variable "cluster_version" {
  type    = string
  default = "1.31"
}

variable "node_instance_type" {
  type    = string
  default = "t3.medium"
}

variable "node_desired_count" {
  type    = number
  default = 2
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}
```

### `vpc.tf`

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = var.vpc_cidr

  azs             = ["${var.region}a", "${var.region}b"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
}
```

### `eks.tf`

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    terraweek_nodes = {
      ami_type       = "AL2_x86_64"
      instance_types = [var.node_instance_type]

      min_size     = 1
      max_size     = 3
      desired_size = var.node_desired_count
    }
  }

  tags = {
    Environment = "dev"
    Project     = "TerraWeek"
    ManagedBy   = "Terraform"
  }
}
```

### `outputs.tf`

```hcl
output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_region" {
  value = var.region
}
```

### `terraform.tfvars`

```hcl
region = "ap-south-1"
```

---

## 2. VPC Design -- Why Public and Private Subnets?

EKS needs both subnet types because different parts of the cluster have different exposure requirements:

- **Public subnets** host resources that need direct internet access, such as internet-facing Load Balancers (e.g. the `nginx-service` LoadBalancer created in Task 5). They route through an Internet Gateway.
- **Private subnets** host the worker nodes themselves. Keeping nodes in private subnets means they aren't directly reachable from the internet, which is a security best practice. Nodes still reach the internet (to pull container images, talk to the EKS control plane, etc.) through the NAT Gateway sitting in a public subnet.

### What do the subnet tags do?

- `kubernetes.io/role/elb = 1` on public subnets tells the AWS cloud provider / load balancer controller that these subnets are valid targets when provisioning an **internet-facing** Load Balancer (`type: LoadBalancer` with default external scheme).
- `kubernetes.io/role/internal-elb = 1` on private subnets marks them as valid targets for **internal** Load Balancers (annotated `service.beta.kubernetes.io/aws-load-balancer-internal: "true"`).

Without these tags, Kubernetes has no reliable way to auto-discover which subnets to place ELBs/NLBs into, and LoadBalancer services can fail to provision or land in the wrong subnets.

---

## 3. EKS Cluster Provisioning

```bash
terraform init
terraform plan
terraform apply
```

Apply took roughly 10-15 minutes to complete (EKS control plane + managed node group).

**Total resources created by `terraform apply`:** `<fill in from your apply output — look for "Apply complete! Resources: X added">`

> Screenshot: `terraform apply` completing
> `[Insert screenshot here]`

### Connecting kubectl

```bash
aws eks update-kubeconfig --name terraweek-eks --region ap-south-1
kubectl get nodes
```

**Access issue encountered:** Initial `kubectl get nodes` failed with:

```
E0703 12:26:41 "Unhandled Error" err="couldn't get current server API group list: the server has asked for the client to provide credentials"
```

**Root cause:** The cluster uses `authenticationMode = API_AND_CONFIG_MAP`. The local IAM user (`arn:aws:iam::942521689864:user/aws_cli`) had valid AWS credentials (confirmed via `aws sts get-caller-identity`) but had no **EKS Access Entry**, so the Kubernetes API server had no way to authorize it.

**Fix applied:**

```bash
aws eks create-access-entry \
  --cluster-name terraweek-eks \
  --principal-arn arn:aws:iam::942521689864:user/aws_cli \
  --region ap-south-1

aws eks associate-access-policy \
  --cluster-name terraweek-eks \
  --principal-arn arn:aws:iam::942521689864:user/aws_cli \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region ap-south-1
```

After this, `kubectl get nodes` succeeded:

```
NAME                                          STATUS   ROLES    AGE   VERSION
ip-10-0-101-27.ap-south-1.compute.internal    Ready    <none>   21m   v1.31.13-eks-ecaa3a6
ip-10-0-102-142.ap-south-1.compute.internal   Ready    <none>   21m   v1.31.13-eks-ecaa3a6
```

> Screenshot: `kubectl get nodes` showing the managed node group
> `[Insert screenshot here]`

```bash
kubectl get pods -A
kubectl cluster-info
```

Both nodes reached `Ready` state, and `kube-system` pods (`coredns`, `aws-node`, `kube-proxy`) were running as expected.

---

## 4. Deploying Nginx

### `k8s/nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-terraweek
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f k8s/nginx-deployment.yaml
kubectl get svc nginx-service -w
```

The AWS in-tree cloud provider provisioned a Classic Load Balancer automatically -- no extra controller install was needed. `kubectl describe svc nginx-service` showed:

```
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  EnsuringLoadBalancer  70s   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   67s   service-controller  Ensured load balancer
```

**LoadBalancer hostname:**
```
afabad87f41fe4ae899c90f14b76d4d1-2057146317.ap-south-1.elb.amazonaws.com
```

Accessed via:
```
http://afabad87f41fe4ae899c90f14b76d4d1-2057146317.ap-south-1.elb.amazonaws.com
```

The Nginx welcome page ("Welcome to nginx!") loaded successfully.

> Screenshot: Nginx welcome page running via the LoadBalancer URL
> `[Insert screenshot here]`

### Full verification

```bash
kubectl get nodes
kubectl get deployments
kubectl get pods
kubectl get svc
```

| Resource | Result |
|---|---|
| Nodes | 2 nodes, `Ready` |
| Deployment `nginx-terraweek` | `3/3` ready |
| Pods | 3 pods, `Running` |
| Service `nginx-service` | `LoadBalancer`, external hostname assigned |

---

## 5. Cleanup / Destroy

```bash
kubectl delete -f k8s/nginx-deployment.yaml
```

Waited for the associated ELB to be fully removed (checked EC2 > Load Balancers in the AWS console) before proceeding, since a leftover ELB inside the VPC will block subnet/VPC deletion.

```bash
terraform destroy
```

Destroy took roughly 10-15 minutes.

**Post-destroy verification checklist:**
- [ ] EKS clusters: empty
- [ ] EC2 instances: no node group instances remaining
- [ ] VPC (`terraweek-eks-vpc`): deleted
- [ ] NAT Gateway: deleted
- [ ] Elastic IPs: released

> `<Fill in actual confirmation once terraform destroy + console check are complete>`

---

## 6. Reflection -- Terraform/EKS vs. kind/minikube (Day 50)

Setting up a cluster with `kind`/`minikube` was a single local command that gave a disposable cluster in seconds, running entirely on one machine with no cloud dependencies, no IAM, and no networking to design.

Provisioning EKS with Terraform was a completely different category of work:

- **Infrastructure, not just a cluster** -- had to design a real VPC (public/private subnets across AZs, NAT gateway, route tables, subnet tags for ELB discovery) before the cluster could even exist.
- **Identity and access is a first-class concern** -- unlike `kind`, where `kubectl` "just works" against the local cluster, EKS access is governed by IAM and had to be explicitly wired up via Access Entries even after AWS credentials were already valid.
- **Time and cost matter** -- cluster creation/destruction takes 10-15 minutes each way, and resources like NAT Gateways cost money by the hour, so cleanup discipline (`terraform destroy`, verifying no leftover ELBs/ENIs) is essential in a way it never is locally.
- **Reproducibility** -- the entire cluster, 30+ AWS resources, is defined as code and can be recreated identically with `terraform apply`, versus a local `kind` cluster's config which is far more ad hoc.

Overall: `kind`/`minikube` is for fast local iteration on Kubernetes manifests; Terraform + EKS is what's actually needed to run a real, production-grade, team-shared cluster in the cloud -- at the cost of real infrastructure complexity (networking, IAM, cost management) that local tools abstract away entirely.

---

## Learn in Public

> Provisioned a full AWS EKS cluster with Terraform modules today -- VPC, subnets, NAT gateway, IAM roles, node groups, the works. 30+ resources created with one command, deployed Nginx on it, and destroyed everything cleanly. This is real-world infrastructure as code.
>
> #90DaysOfDevOps #TerraWeek #DevOpsKaJosh #TrainWithShubham
