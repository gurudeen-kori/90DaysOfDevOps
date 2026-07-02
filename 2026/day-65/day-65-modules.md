# Day 65 — Terraform Modules: Build Reusable Infrastructure

## 1. Module Structure (Root vs. Child Modules)

Project layout:

```
terraform-modules/
  main.tf                    # Root module -- calls child modules
  variables.tf                # Root variables
  outputs.tf                  # Root outputs
  providers.tf                 # Provider config
  modules/
    ec2-instance/
      main.tf                # EC2 resource definition
      variables.tf            # Module inputs
      outputs.tf               # Module outputs
    security-group/
      main.tf                # Security group resource definition
      variables.tf            # Module inputs
      outputs.tf               # Module outputs
```

**Root module vs. child module:**

- The **root module** is whatever directory you run `terraform init` / `plan` / `apply` in — here, `terraform-modules/`. It's the entry point of the configuration. Terraform reads all `.tf` files in that directory and starts building the dependency graph from there.
- A **child module** is any module called via a `module` block (`./modules/ec2-instance`, `./modules/security-group`, or a registry module like `terraform-aws-modules/vpc/aws`). Child modules don't run on their own — they're only instantiated when something (the root module, or another module) calls them with `source = "..."`.
- Practically: the root module owns provider configuration and is the only place `terraform apply` acts directly on. Child modules are reusable, parameterized building blocks — the same `ec2-instance` module gets called twice in this project (`web_server` and `api_server`) with different inputs, producing two independent instances from one set of `.tf` files.

## 2. Custom EC2 Module (`modules/ec2-instance/`)

**variables.tf**
```hcl
variable "ami_id" {
  description = "The AMI ID to use for the instance"
  type        = string
}

variable "instance_type" {
  description = "The EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "subnet_id" {
  description = "The subnet ID to launch the instance in"
  type        = string
}

variable "security_group_ids" {
  description = "List of security group IDs to attach to the instance"
  type        = list(string)
}

variable "instance_name" {
  description = "The value for the instance's Name tag"
  type        = string
}

variable "tags" {
  description = "Additional tags to apply to the instance"
  type        = map(string)
  default     = {}
}
```

**main.tf**
```hcl
resource "aws_instance" "this" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = merge(
    {
      Name = var.instance_name
    },
    var.tags
  )
}
```

**outputs.tf**
```hcl
output "instance_id" {
  description = "The ID of the EC2 instance"
  value       = aws_instance.this.id
}

output "public_ip" {
  description = "The public IP address of the EC2 instance"
  value       = aws_instance.this.public_ip
}

output "private_ip" {
  description = "The private IP address of the EC2 instance"
  value       = aws_instance.this.private_ip
}
```

The companion **`modules/security-group/`** module follows the same pattern, plus a `dynamic "ingress"` block that loops over `var.ingress_ports` (default `[22, 80]`) to generate one `ingress {}` block per port, and a single `egress {}` block allowing all outbound traffic.

## 3. Root `main.tf` — Calling Custom and Registry Modules Together

```hcl
locals {
  common_tags = {
    Project     = "TerraWeek"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Registry module (Task 5) — replaces a hand-written VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "terraweek-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway   = false
  enable_dns_hostnames = true

  tags = local.common_tags
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Custom module (Task 3)
module "web_sg" {
  source = "./modules/security-group"

  vpc_id        = module.vpc.vpc_id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = local.common_tags
}

# Custom module (Task 2), called twice with different inputs
module "web_server" {
  source = "./modules/ec2-instance"

  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = module.vpc.public_subnets[0]
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-web"
  tags               = local.common_tags
}

module "api_server" {
  source = "./modules/ec2-instance"

  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = module.vpc.public_subnets[0]
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-api"
  tags               = local.common_tags
}
```

Root `outputs.tf` surfaces the module outputs one level up:

```hcl
output "web_server_ip" {
  value = module.web_server.public_ip
}

output "api_server_ip" {
  value = module.api_server.public_ip
}
```

Notice the pattern: `./modules/...` sources are **local** paths (relative to the calling module), while `terraform-aws-modules/vpc/aws` is a **registry** source (`<namespace>/<name>/<provider>`), pulled from `registry.terraform.io` and pinned with `version = "~> 5.0"`.

## 4. Applying and Verifying (Task 4)

> This step wasn't run in this environment (no AWS credentials configured / no network path to AWS or the Terraform Registry). Run these yourself:

```bash
terraform init    # downloads/links local + registry modules
terraform plan    # should show ~20+ resources: VPC module, 2x SG rules, 2x EC2, etc.
terraform apply
```

Once applied, `terraform state list` will show every resource prefixed by its module path:

```
module.vpc.aws_vpc.this[0]
module.vpc.aws_subnet.public[0]
module.vpc.aws_subnet.public[1]
module.web_sg.aws_security_group.this
module.web_server.aws_instance.this
module.api_server.aws_instance.this
```

**Screenshot placeholder:** after `terraform apply` succeeds, take a screenshot of the EC2 console showing `terraweek-web` and `terraweek-api` running in the same security group, and drop it in this section before committing.

## 5. Hand-Written VPC vs. Registry VPC Module (Task 5)

| | Hand-written (Day 62 style) | `terraform-aws-modules/vpc/aws` |
|---|---|---|
| Resources you write | `aws_vpc`, `aws_subnet` (x N), `aws_internet_gateway`, `aws_route_table`, `aws_route_table_association` (x N), `aws_route` — typically 8–12 resource blocks hand-maintained | One `module` block, ~10 lines of config |
| Resources actually created | Same count, but you own every argument and every edge case (NAT, DNS, tagging conventions, multi-AZ symmetry) | The module creates the equivalent resource set (and more, if `enable_nat_gateway`, VPN, flow logs, etc. are turned on) — commonly 15–25+ resources once public/private subnets, route tables, and associations across 2 AZs are counted |
| Maintenance | You must repeat all of this per project, and keep it consistent | Versioned, tested by the community, upgraded with `terraform init -upgrade` |
| Where downloaded | N/A — it's your own code | `.terraform/modules/` — check `.terraform/modules/modules.json` to see the resolved source and version, and the actual module source is cached under `.terraform/modules/vpc/` |

Net effect: the registry module trades a small amount of control (you accept its opinions about naming, tagging, and structure) for a large reduction in code you have to write and maintain — and it comes with someone else's testing and edge-case handling built in.

## 6. Five Module Best Practices

1. **Always pin versions for registry modules.** An unpinned `source` will happily pull in breaking changes on the next `init -upgrade`. `version = "~> 5.0"` (or an exact version for production) keeps upgrades intentional instead of accidental.
2. **Keep modules focused — one concern per module.** The EC2 module only knows about EC2; the security-group module only knows about security groups. Composing small modules together is easier to reason about than one module that tries to do networking, compute, and IAM all at once.
3. **Use variables for everything, hardcode nothing.** Every value that could plausibly differ between callers — AMI, instance type, ports, tags — is a variable with a sensible default where one makes sense. This is what lets the same `ec2-instance` module stand up both `web_server` and `api_server` with different inputs.
4. **Always define outputs so callers can reference resources.** Without `outputs.tf`, a module is a black box — the root module (or another module) has no way to wire its results (like `sg_id` or `public_ip`) into anything downstream.
5. **Add a `README.md` to every custom module.** Registry modules document every input/output on `registry.terraform.io`; your own modules should do the same locally so six months from now (or a teammate today) doesn't have to open `variables.tf` just to remember what `ingress_ports` does.
