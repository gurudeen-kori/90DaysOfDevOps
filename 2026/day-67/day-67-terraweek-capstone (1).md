# Day 67 — TerraWeek Capstone: Multi-Environment Infrastructure with Workspaces and Modules

## Task 1: Terraform Workspaces — Answers

**What does `terraform.workspace` return inside a config?**
It returns the name of the currently selected workspace as a string (e.g. `"dev"`, `"staging"`, `"prod"`, or `"default"`). It's a built-in value, available in any `.tf` file without being declared as a variable, and it updates automatically whenever you run `terraform workspace select <name>`.

**Where does each workspace store its state file?**
With the local backend, each non-default workspace gets its own state file at:
```
terraform.tfstate.d/<workspace>/terraform.tfstate
```
The `default` workspace still uses the plain `terraform.tfstate` in the project root. With a remote backend (e.g. S3), the key is namespaced instead: `env:/<workspace>/<key>`.

**How is this different from using separate directories per environment?**
- **Workspaces**: one copy of the code, multiple state files, switch context with `terraform workspace select`. Easy to keep environments in sync because there's only one codebase to update — but it's also easy to apply to the wrong workspace by accident since the code looks identical everywhere.
- **Separate directories**: one full copy of the code (or a shared module) per environment, each with its own backend config and state. More boilerplate, but every environment is explicit and isolated on disk — you can't "forget" which one you're in, and you can diverge dev from prod deliberately without conditional logic.

In practice, workspaces are best for environments that are structurally identical and differ only by input values (this project). Separate directories are better once environments need materially different resources or backend accounts.

---

## Note on Resource Naming

The Terraform resource labels and `Name` tags in this project were updated per request:

| Resource | Resource label | `Name` tag |
|---|---|---|
| `aws_vpc` | `my_vpc` | `my_vpc` |
| `aws_subnet` | `my_subnet` | `my_subnet` |
| `aws_internet_gateway` | `my_vpc_igw` | `my_vpc_igw` |
| `aws_security_group` | `my_security_group` | `my_security_group` |
| `aws_instance` | `my_instance` | `my_instance` |

**Heads up:** because these names are now fixed strings rather than workspace-derived (e.g. `${var.project_name}-${var.environment}-vpc`), all three environments will show the *same* `Name` tag in the AWS console — three VPCs will each say `my_vpc`, three instances will each say `my_instance`, and so on. This is a deviation from the `<project>-<environment>-<resource>` naming pattern recommended in the Task 6 best practices guide below, since you'll now need to rely on the `Environment` tag (or the AWS account/region/workspace context) to tell dev, staging, and prod resources apart rather than the name alone. If you want the isolation-by-naming benefit back, add `-${var.environment}` back into the `Name` tag values while keeping the Terraform resource labels as `my_vpc`, `my_instance`, etc.

## Project Structure

```
terraweek-capstone/
├── main.tf                   # Root module -- calls child modules
├── variables.tf               # Root variables
├── outputs.tf                  # Root outputs
├── providers.tf                # AWS provider and S3 backend
├── locals.tf                   # Local values derived from terraform.workspace
├── dev.tfvars                  # Dev environment values
├── staging.tfvars              # Staging environment values
├── prod.tfvars                 # Prod environment values
├── .gitignore                  # Ignore state, .terraform, tfvars
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── security-group/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ec2-instance/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Why is this file structure best practice?

- **Separation of concerns** — `providers.tf`, `variables.tf`, `outputs.tf`, `main.tf`, and `locals.tf` each have one job. Anyone opening the repo knows exactly where to look for provider config vs. resource logic vs. inputs/outputs, instead of hunting through one giant file.
- **Modules encapsulate infrastructure concerns** — the VPC, security group, and EC2 logic are each self-contained with their own inputs and outputs, so they can be tested, versioned, and reused independently (even in other projects).
- **tfvars per environment** — keeps environment-specific values out of the core logic entirely. The same `main.tf` never changes between dev, staging, and prod; only the values fed into it do.
- **`.gitignore` protects state and secrets** — Terraform state can contain sensitive data (IPs, resource IDs, sometimes secrets in outputs), and tfvars files can contain credentials or account-specific values. Neither belongs in version control.
- **Predictability at scale** — this layout is the de facto standard across infrastructure teams, so a new engineer can clone the repo and immediately understand it without a walkthrough.

---

## Custom Modules

### Module 1: `modules/vpc`

**`main.tf`**
```hcl
resource "aws_vpc" "my_vpc" {
  cidr_block           = var.cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "my_vpc"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_subnet" "my_subnet" {
  vpc_id                  = aws_vpc.my_vpc.id
  cidr_block              = var.public_subnet_cidr
  map_public_ip_on_launch = true

  tags = {
    Name        = "my_subnet"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_internet_gateway" "my_vpc_igw" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name        = "my_vpc_igw"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my_vpc_igw.id
  }

  tags = {
    Name        = "${var.project_name}-${var.environment}-public-rt"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.my_subnet.id
  route_table_id = aws_route_table.public.id
}
```

**`variables.tf`**
```hcl
variable "cidr" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "public_subnet_cidr" {
  description = "CIDR block for the public subnet"
  type        = string
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
}

variable "project_name" {
  description = "Project name used for tagging and naming"
  type        = string
}
```

**`outputs.tf`**
```hcl
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.my_vpc.id
}

output "subnet_id" {
  description = "ID of the created public subnet"
  value       = aws_subnet.my_subnet.id
}
```

### Module 2: `modules/security-group`

**`main.tf`**
```hcl
resource "aws_security_group" "my_security_group" {
  name        = "my_security_group"
  description = "Security group for ${var.project_name} ${var.environment}"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      description = "Allow inbound on port ${ingress.value}"
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "my_security_group"
    Environment = var.environment
    Project     = var.project_name
  }
}
```

**`variables.tf`**
```hcl
variable "vpc_id" {
  description = "ID of the VPC to attach the security group to"
  type        = string
}

variable "ingress_ports" {
  description = "List of TCP ports to allow inbound"
  type        = list(number)
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
}

variable "project_name" {
  description = "Project name used for tagging and naming"
  type        = string
}
```

**`outputs.tf`**
```hcl
output "sg_id" {
  description = "ID of the created security group"
  value       = aws_security_group.my_security_group.id
}
```

### Module 3: `modules/ec2-instance`

**`main.tf`**
```hcl
resource "aws_instance" "my_instance" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = {
    Name        = "my_instance"
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
}
```

**`variables.tf`**
```hcl
variable "ami_id" {
  description = "AMI ID to launch the instance from"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
}

variable "subnet_id" {
  description = "Subnet ID to launch the instance into"
  type        = string
}

variable "security_group_ids" {
  description = "List of security group IDs to attach"
  type        = list(string)
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
}

variable "project_name" {
  description = "Project name used for tagging and naming"
  type        = string
}
```

**`outputs.tf`**
```hcl
output "instance_id" {
  description = "ID of the created EC2 instance"
  value       = aws_instance.my_instance.id
}

output "public_ip" {
  description = "Public IP address of the created EC2 instance"
  value       = aws_instance.my_instance.public_ip
}
```

---

## Root `main.tf` — Workspace-Aware Module Calls

```hcl
# Root module -- calls the vpc, security-group, and ec2-instance child modules.
# terraform.workspace (via local.environment) drives naming and tagging so the
# exact same code produces isolated, correctly-labeled infrastructure for
# dev, staging, and prod.

module "vpc" {
  source = "./modules/vpc"

  cidr               = var.vpc_cidr
  public_subnet_cidr = var.subnet_cidr
  environment        = local.environment
  project_name       = var.project_name
}

module "security_group" {
  source = "./modules/security-group"

  vpc_id        = module.vpc.vpc_id
  ingress_ports = var.ingress_ports
  environment   = local.environment
  project_name  = var.project_name
}

module "ec2_instance" {
  source = "./modules/ec2-instance"

  ami_id             = var.ami_id
  instance_type      = var.instance_type
  subnet_id          = module.vpc.subnet_id
  security_group_ids = [module.security_group.sg_id]
  environment        = local.environment
  project_name       = var.project_name
}
```

Supporting `locals.tf`:

```hcl
locals {
  environment = terraform.workspace
  name_prefix = "${var.project_name}-${local.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = local.environment
    ManagedBy   = "Terraform"
    Workspace   = terraform.workspace
  }
}
```

No `if terraform.workspace == "dev"` branching anywhere — the workspace name flows straight into `local.environment`, which flows into every module call. All environment-specific *values* (CIDR, instance size, ports) come from tfvars, not from conditionals in the code.

---

## Environment tfvars — Differences Highlighted

| Setting | dev | staging | prod |
|---|---|---|---|
| `vpc_cidr` | `10.0.0.0/16` | `10.1.0.0/16` | `10.2.0.0/16` |
| `subnet_cidr` | `10.0.1.0/24` | `10.1.1.0/24` | `10.2.1.0/24` |
| `instance_type` | `t2.micro` | `t2.small` | `t3.small` |
| `ingress_ports` | `[22, 80]` (SSH open) | `[22, 80, 443]` (SSH open) | `[80, 443]` (**no SSH**) |

**`dev.tfvars`**
```hcl
vpc_cidr      = "10.0.0.0/16"
subnet_cidr   = "10.0.1.0/24"
instance_type = "t2.micro"
ingress_ports = [22, 80]
```

**`staging.tfvars`**
```hcl
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
instance_type = "t2.small"
ingress_ports = [22, 80, 443]
```

**`prod.tfvars`**
```hcl
vpc_cidr      = "10.2.0.0/16"
subnet_cidr   = "10.2.1.0/24"
instance_type = "t3.small"
ingress_ports = [80, 443]
```

**Why these differences matter:**
- **Non-overlapping CIDRs** mean the three VPCs could theoretically be peered later without conflict, and there's zero chance of confusing one environment's address space for another's.
- **Instance types scale up** with the environment's expected load — dev is cheap and disposable, prod gets more headroom.
- **Prod drops port 22** — no direct SSH into production instances. Access should go through Session Manager, a bastion, or similar, not an open SSH port. Dev and staging keep SSH open for day-to-day debugging.

---

## Deployment & Verification

```bash
# Dev
terraform workspace select dev
terraform plan  -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"

# Staging
terraform workspace select staging
terraform plan  -var-file="staging.tfvars"
terraform apply -var-file="staging.tfvars"

# Prod
terraform workspace select prod
terraform plan  -var-file="prod.tfvars"
terraform apply -var-file="prod.tfvars"

# Verify outputs per workspace
terraform workspace select dev     && terraform output
terraform workspace select staging && terraform output
terraform workspace select prod    && terraform output
```

**AWS console checklist:**
- [ ] Three separate VPCs, each with a distinct CIDR range, each tagged `Name = my_vpc`
- [ ] Three subnets, each tagged `Name = my_subnet`
- [ ] Three internet gateways, each tagged `Name = my_vpc_igw`
- [ ] Three security groups, each tagged `Name = my_security_group`
- [ ] Three EC2 instances, each tagged `Name = my_instance`, each with the correct instance type for its environment
- [ ] The `Environment` tag on every resource is the only thing distinguishing dev / staging / prod resources apart, since the `Name` tag is now fixed across environments — use `Environment` (or the workspace) to tell them apart in the console rather than the `Name` tag

*(Insert screenshot: all three environments running simultaneously in AWS)*
*(Insert screenshot: `terraform output` for each workspace)*

**Isolation check:** yes — each workspace has its own state file, its own VPC (non-overlapping CIDR blocks, no peering configured), its own subnet, its own security group, and its own EC2 instance. Nothing is shared between environments except the Terraform code itself and the remote backend bucket/table.

---

## Task 6: Terraform Best Practices Guide

**File structure** — Separate `providers.tf`, `variables.tf`, `outputs.tf`, `main.tf`, and `locals.tf`. Never dump everything into one `main.tf`; a reader should be able to guess which file holds what before opening it.

**State management** — Always use a remote backend (S3 + DynamoDB, Terraform Cloud, etc.), never local state for anything beyond a personal sandbox. Enable state locking so two people (or two CI runs) can't corrupt state by applying simultaneously. Enable versioning on the backend bucket so a bad state file can be rolled back.

**Variables** — Never hardcode environment-specific values into resources. Use a `.tfvars` file per environment, and add `validation` blocks on variables where invalid input could cause expensive or dangerous mistakes (e.g. rejecting a `/8` CIDR by accident).

**Modules** — One concern per module (VPC, security group, compute — not one giant "infra" module). Always define explicit `variables.tf` and `outputs.tf` — a module's contract should be readable without opening `main.tf`. Pin versions on registry modules so an upstream update doesn't silently change your infrastructure.

**Workspaces** — Use workspaces when environments are structurally identical and differ only by input values. Reference `terraform.workspace` in `locals.tf` rather than scattering it through the codebase, so there's one place that translates "workspace" into "environment."

**Security** — `.gitignore` state files, `.terraform/`, and tfvars. Encrypt state at rest (S3 server-side encryption or backend-native encryption). Restrict who/what can read or write the backend — state can leak resource IDs, IPs, and sometimes secrets.

**Commands** — Always run `terraform plan` before `apply` and actually read the diff. Run `terraform fmt` and `terraform validate` before committing — catch syntax and style issues before they hit a PR.

**Tagging** — Tag every resource with at least `Project`, `Environment`, and `ManagedBy`. This is what makes cost allocation, cleanup scripts, and "whose resource is this" questions answerable at 2am.

**Naming** — Consistent prefix pattern: `<project>-<environment>-<resource>` (e.g. `terraweek-prod-server`). Predictable names make console debugging and automation far easier.

**Cleanup** — Always `terraform destroy` non-production environments when they're not actively in use. Idle dev/staging infrastructure is pure cost with no benefit.

---

## TerraWeek: Day-by-Day Concept Map

| Day | Concepts |
|---|---|
| 61 | IaC, HCL, init/plan/apply/destroy, state basics |
| 62 | Providers, resources, dependencies, lifecycle |
| 63 | Variables, outputs, data sources, locals, functions |
| 64 | Remote backend, locking, import, drift |
| 65 | Custom modules, registry modules, versioning |
| 66 | EKS with modules, real-world provisioning |
| 67 | Workspaces, multi-env, capstone project |

---

## Cleanup

```bash
terraform workspace select prod
terraform destroy -var-file="prod.tfvars"

terraform workspace select staging
terraform destroy -var-file="staging.tfvars"

terraform workspace select dev
terraform destroy -var-file="dev.tfvars"

terraform workspace select default
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod
```

Confirmed clean in the AWS console — no leftover VPCs, instances, security groups, or internet gateways in any region used.

#90DaysOfDevOps #TerraWeek #DevOpsKaJosh #TrainWithShubham
