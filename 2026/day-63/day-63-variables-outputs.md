# Day 63 — Variables, Outputs, Data Sources and Expressions

Refactored the Day 62 Terraform config to remove all hardcoded values — region, CIDR blocks, AMI IDs, instance types, and tags are now fully parameterized and environment-aware.

---

## Task 1: Extracted Variables

**`variables.tf`**

```hcl
variable "region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "ap-south-1"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_cidr" {
  description = "CIDR block for the public subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "project_name" {
  description = "Name of the project (required)"
  type        = string
  # no default -> Terraform will prompt for this value
}

variable "environment" {
  description = "Deployment environment (dev/staging/prod)"
  type        = string
  default     = "dev"
}

variable "allowed_ports" {
  description = "List of ports allowed through the security group"
  type        = list(number)
  default     = [22, 80, 443]
}

variable "extra_tags" {
  description = "Additional tags to merge into every resource"
  type        = map(string)
  default     = {}
}
```

Running `terraform plan` without a value for `project_name` correctly prompts:

```
var.project_name
  Name of the project (required)

  Enter a value:
```

### Five Variable Types in Terraform

| Type | Description | Example |
|---|---|---|
| `string` | A single text value | `"terraweek"` |
| `number` | An integer or float | `22`, `3.14` |
| `bool` | True/false flag | `true`, `false` |
| `list(type)` | An ordered collection of values, all the same type | `[22, 80, 443]` |
| `map(type)` | A set of key-value pairs, all values the same type | `{ dev = "t2.micro", prod = "t3.small" }` |

(Terraform also supports complex structural types — `object()`, `tuple()`, `set()` — built from these five primitives.)

---

## Task 2: Variable Files and Precedence

**`terraform.tfvars`** (loaded automatically)

```hcl
project_name  = "terraweek"
environment   = "dev"
instance_type = "t2.micro"
```

**`prod.tfvars`** (loaded explicitly via `-var-file`)

```hcl
project_name  = "terraweek"
environment   = "prod"
instance_type = "t3.small"
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
```

Tested:

```bash
terraform plan                                # uses terraform.tfvars automatically
terraform plan -var-file="prod.tfvars"        # uses prod.tfvars
terraform plan -var="instance_type=t2.nano"   # CLI override
export TF_VAR_environment="staging"
terraform plan                                # terraform.tfvars still wins -> environment stays "dev"
```

The last test is the important one: even with `TF_VAR_environment=staging` exported, the plan kept `environment = "dev"` because `terraform.tfvars` was present and takes priority over environment variables.

### Variable Precedence (lowest → highest priority, verified)

1. **Variable `default` value** (in `variables.tf`)
2. **Environment variables** (`TF_VAR_*`)
3. **`terraform.tfvars`** (auto-loaded)
4. **`*.auto.tfvars` / `*.auto.tfvars.json`** files (alphabetical order)
5. **`-var-file` flag** (e.g. `prod.tfvars`)
6. **`-var` flag on the CLI** — highest priority, overrides everything

> Note: this is the order confirmed by actual behavior in this project (env var was overridden by `terraform.tfvars`), matching official Terraform documentation. CLI `-var` always wins over any file-based value.

---

## Task 3: Outputs

**`outputs.tf`**

```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.my_vpc.id
}

output "subnet_id" {
  description = "The ID of the public subnet"
  value       = aws_subnet.my-subnet.id
}

output "instance_id" {
  description = "The ID of the EC2 instance"
  value       = aws_instance.my-instance.id
}

output "instance_public_ip" {
  description = "The public IP address of the EC2 instance"
  value       = aws_instance.my-instance.public_ip
}

output "instance_public_dns" {
  description = "The public DNS name of the EC2 instance"
  value       = aws_instance.my-instance.public_dns
}

output "security_group_id" {
  description = "The ID of the security group"
  value       = aws_security_group.my_sg.id
}
```

After `terraform apply`, outputs printed automatically:

```
Outputs:

instance_id          = "i-0ae4cbe7903948073"
instance_public_dns  = "ec2-xx-xxx-xx-xx.ap-south-1.compute.amazonaws.com"
instance_public_ip   = "xx.xxx.xx.xx"
security_group_id    = "sg-056c869c466f9d571"
subnet_id            = "subnet-040b7e0ac32620894"
vpc_id               = "vpc-03949a8e1233c6316"
```

Verified `terraform output instance_public_ip` returns the correct, current IP — matched the value shown after `apply` completed, and confirmed via `terraform state show aws_instance.my-instance`.

```bash
terraform output                    # all outputs
terraform output instance_public_ip # single output
terraform output -json              # JSON format
```

*(Insert screenshot of `terraform output` here for submission.)*

---

## Task 4: Data Sources

**`data.tf`**

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}
```

`aws_instance` now references `data.aws_ami.amazon_linux.id` instead of a hardcoded AMI, and `aws_subnet` uses `data.aws_availability_zones.available.names[0]` instead of a hardcoded AZ.

Verified with:

```bash
terraform console
> data.aws_ami.amazon_linux.id
> data.aws_availability_zones.available.names
```

Both resolved correctly without any hardcoded value in the config — confirming the same code would work unmodified if the `region` variable were changed to any other AWS region.

### Resource vs Data Source

| | `resource` | `data` |
|---|---|---|
| Purpose | Creates and manages infrastructure | Reads existing/external information |
| Lifecycle | Create → Update → Destroy, fully owned by Terraform | Read-only, re-evaluated on every plan/apply |
| Effect on `terraform destroy` | Gets destroyed | Nothing happens — it was never created |
| Example | `aws_instance`, `aws_vpc`, `aws_security_group` | `aws_ami`, `aws_availability_zones`, `aws_caller_identity` |

A `resource` block tells Terraform "build and own this." A `data` block tells Terraform "go look this up for me" — it never creates, modifies, or deletes anything in AWS.

---

## Task 5: Locals for Dynamic Values

**`locals.tf`**

```hcl
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

Applied across all resources:

```hcl
tags = merge(local.common_tags, {
  Name = "${local.name_prefix}-vpc"
})
```

(and equivalently `-subnet`, `-server`, `-sg` for the other resources)

After `terraform apply`, every resource in the AWS console carries the same `Project`, `Environment`, and `ManagedBy` tags, plus its own unique `Name` tag — confirmed consistent tagging across VPC, Subnet, EC2 instance, and Security Group.

---

## Task 6: Built-in Functions and Conditional Expressions

Practiced in `terraform console`:

```
> upper("terraweek")
"TERRAWEEK"

> join("-", ["terra", "week", "2026"])
"terra-week-2026"

> format("arn:aws:s3:::%s", "my-bucket")
"arn:aws:s3:::my-bucket"

> length(["a", "b", "c"])
3

> lookup({dev = "t2.micro", prod = "t3.small"}, "dev")
"t2.micro"

> toset(["a", "b", "a"])
toset(["a", "b"])

> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
```

Added the conditional expression to `aws_instance`:

```hcl
instance_type = var.environment == "prod" ? "t3.small" : "t2.micro"
```

Applied with `environment = "prod"` (via `prod.tfvars`) and confirmed the plan showed `instance_type` changing from `t2.micro` to `t3.small` automatically, with no manual override needed.

### Five Most Useful Functions

1. **`merge()`** — Combines two or more maps into one; later maps override matching keys. Used to merge `local.common_tags` with resource-specific `Name` tags so every resource gets consistent tagging without repeating the same key-value pairs everywhere.

2. **`lookup()`** — Retrieves a value from a map by key, with a safe fallback default if the key doesn't exist. Useful for environment-based config tables (e.g. instance sizes per environment) without the risk of a hard failure on a missing key.

3. **`cidrsubnet()`** — Calculates a subnet CIDR block from a parent CIDR, given a newbit count and subnet index. Removes manual IP-math errors when carving multiple subnets out of a VPC.

4. **`format()`** — Builds strings from a template with placeholders (`%s`, `%d`). Useful for constructing ARNs, resource names, or any pattern-based string in a readable, single line.

5. **`toset()`** — Removes duplicate values from a list and converts it to a set. Particularly useful before passing a list into `for_each`, which requires a set rather than a list with potential duplicates.

### Difference Between Variable, Local, Output, and Data

| Concept | Purpose |
|---|---|
| **`variable`** | An input — a value supplied from outside the config (tfvars, CLI, env var, or default). Defines what the user *can* configure. |
| **`local`** | A computed/derived value defined inside the config, used to avoid repeating expressions. Not an input, not directly settable from outside. |
| **`output`** | An exported value from the config, shown after `apply` or queried with `terraform output`. Used to expose resource attributes (IDs, IPs) to the user or to other Terraform configs/modules. |
| **`data`** | A read-only lookup of existing information (from the provider or external source), used to avoid hardcoding values like AMI IDs or AZ names. |

In short: `variable` goes in, `data` comes from outside (read-only), `local` is computed in the middle, `output` goes out.

---

## Summary

All hardcoded values — region, CIDRs, AMI ID, instance type, AZ, tags — have been replaced with variables, data sources, and locals. The same configuration now works across environments (`dev`/`prod` via `.tfvars`) and across AWS regions (via dynamic AMI/AZ lookup) without any code changes.
