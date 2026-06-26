# Day 62 – Providers, Resources and Dependencies

## Overview

Today I built a complete AWS networking stack using Terraform — a VPC with a subnet, internet gateway, route table, security group, and an EC2 instance. The key learning was understanding how Terraform resolves **dependency order** automatically through implicit references, and how to override that with `depends_on` when needed.

---

## Task 1: Explore the AWS Provider

### `providers.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### What does `~> 5.0` mean?

| Constraint   | Meaning                                                                 |
|--------------|-------------------------------------------------------------------------|
| `~> 5.0`     | Allow any version `>= 5.0` and `< 6.0` (patch and minor upgrades only) |
| `>= 5.0`     | Allow any version 5.0 or higher — including 6.x, 7.x, etc.            |
| `= 5.0.0`    | Pin to **exactly** version 5.0.0, no upgrades at all                   |

`~>` is called the **pessimistic constraint operator**. It's the recommended approach in production: you get bug fixes but avoid breaking changes from major version bumps.

### The `.terraform.lock.hcl` file

After running `terraform init`, Terraform creates `.terraform.lock.hcl`. This file:

- Records the **exact version** of each provider that was installed
- Stores **cryptographic hashes** to verify the provider binary hasn't been tampered with
- Should be **committed to version control** so all team members use the same provider version
- Prevents unexpected upgrades when someone else runs `terraform init`

---

## Task 2: Build a VPC from Scratch

### `main.tf` (networking resources)

```hcl
# ──────────────────────────────────────────
# VPC
# Root of the network — everything lives inside this
# ──────────────────────────────────────────
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"   # 65,536 total IPs

  tags = {
    Name = "TerraWeek-VPC"
  }
}

# ──────────────────────────────────────────
# Subnet
# Implicit dependency: references aws_vpc.main.id
# Terraform will create the VPC first automatically
# ──────────────────────────────────────────
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id   # <-- implicit dependency
  cidr_block              = "10.0.1.0/24"     # 256 IPs
  map_public_ip_on_launch = true              # instances get public IPs by default

  tags = {
    Name = "TerraWeek-Public-Subnet"
  }
}

# ──────────────────────────────────────────
# Internet Gateway
# Allows VPC resources to reach the internet
# Implicit dependency: references aws_vpc.main.id
# ──────────────────────────────────────────
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id   # <-- implicit dependency

  tags = {
    Name = "TerraWeek-IGW"
  }
}

# ──────────────────────────────────────────
# Route Table
# Defines routing rules for the VPC
# Route 0.0.0.0/0 → internet gateway (default route for internet traffic)
# ──────────────────────────────────────────
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id   # <-- implicit dependency

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id   # <-- implicit dependency
  }

  tags = {
    Name = "TerraWeek-Public-RT"
  }
}

# ──────────────────────────────────────────
# Route Table Association
# Links the public subnet to the public route table
# Implicit dependencies: route table + subnet
# ──────────────────────────────────────────
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id        # <-- implicit dependency
  route_table_id = aws_route_table.public.id   # <-- implicit dependency
}
```

**`terraform plan` output:** 5 resources to add ✅

---

## Task 3: Implicit Dependencies

### How Terraform resolves order

When you write `vpc_id = aws_vpc.main.id`, Terraform sees that this resource **references** another resource's attribute. It builds an internal dependency graph and ensures the **referenced resource is created first**.

This is called an **implicit dependency** — you don't have to tell Terraform the order; it figures it out from the references in your code.

### What would happen without the VPC?

If the VPC didn't exist yet, the subnet creation API call would fail immediately — AWS requires a valid VPC ID. Terraform prevents this by always creating the VPC first when it detects the reference.

### All implicit dependencies in this config

| Resource | Depends on | Via attribute |
|---|---|---|
| `aws_subnet.public` | `aws_vpc.main` | `vpc_id` |
| `aws_internet_gateway.main` | `aws_vpc.main` | `vpc_id` |
| `aws_route_table.public` | `aws_vpc.main` | `vpc_id` |
| `aws_route_table.public` | `aws_internet_gateway.main` | `route.gateway_id` |
| `aws_route_table_association.public` | `aws_subnet.public` | `subnet_id` |
| `aws_route_table_association.public` | `aws_route_table.public` | `route_table_id` |
| `aws_instance.main` | `aws_subnet.public` | `subnet_id` |
| `aws_instance.main` | `aws_security_group.main` | `vpc_security_group_ids` |

---

## Task 4: Security Group and EC2 Instance

```hcl
# ──────────────────────────────────────────
# Security Group
# Controls inbound/outbound traffic for the EC2 instance
# ──────────────────────────────────────────
resource "aws_security_group" "main" {
  name        = "TerraWeek-SG"
  description = "Allow SSH and HTTP inbound, all outbound"
  vpc_id      = aws_vpc.main.id   # <-- implicit dependency

  # Allow SSH from anywhere (port 22)
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow HTTP from anywhere (port 80)
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow all outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "TerraWeek-SG"
  }
}

# ──────────────────────────────────────────
# EC2 Instance
# Amazon Linux 2 in the public subnet
# Implicit dependencies: subnet + security group
# ──────────────────────────────────────────
resource "aws_instance" "main" {
  # Amazon Linux 2 AMI (us-east-1) — update for your region
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"

  subnet_id                   = aws_subnet.public.id           # <-- implicit dependency
  vpc_security_group_ids      = [aws_security_group.main.id]  # <-- implicit dependency
  associate_public_ip_address = true

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "TerraWeek-Server"
  }
}
```

---

## Task 5: Explicit Dependencies with `depends_on`

```hcl
# ──────────────────────────────────────────
# S3 Bucket for Application Logs
# No direct reference to the EC2 instance,
# but we want it created AFTER the instance
# is ready — explicit dependency required
# ──────────────────────────────────────────
resource "aws_s3_bucket" "app_logs" {
  bucket = "terraweek-app-logs-${random_id.suffix.hex}"

  depends_on = [aws_instance.main]   # <-- explicit dependency

  tags = {
    Name = "TerraWeek-App-Logs"
  }
}

resource "random_id" "suffix" {
  byte_length = 4
}
```

### Dependency Graph

Run `terraform graph | dot -Tpng > graph.png` (requires Graphviz), or paste the DOT output into [webgraphviz.com](http://www.webgraphviz.com/).

The graph shows a clear tree:

```
aws_vpc.main
├── aws_subnet.public
│   └── aws_route_table_association.public
│       └── aws_route_table.public
└── aws_internet_gateway.main
    └── aws_route_table.public
aws_security_group.main + aws_subnet.public
└── aws_instance.main
    └── aws_s3_bucket.app_logs  (via depends_on)
```

### When would you use `depends_on` in real projects?

1. **IAM role before Lambda**: When a Lambda function's execution role is created and you want to ensure the IAM role's policy attachments are fully propagated before the function is created — even though the Lambda only references the role ARN (not the policy).

2. **RDS instance before application config**: When you store a database connection string in AWS SSM Parameter Store after the RDS instance is created, but the parameter resource has no direct attribute reference to the RDS resource — only its endpoint string. `depends_on = [aws_db_instance.main]` ensures the DB exists first.

---

## Task 6: Lifecycle Rules

```hcl
lifecycle {
  create_before_destroy = true
}
```

When you change the AMI ID and run `terraform plan`, you'll see:

```
# aws_instance.main must be replaced
+/- resource "aws_instance" "main" {
    ...
    Plan: 1 to add, 0 to change, 1 to destroy.
```

With `create_before_destroy = true`, Terraform spins up the **new instance first**, then terminates the old one — minimizing downtime.

### The three lifecycle arguments

| Argument | Purpose | When to use |
|---|---|---|
| `create_before_destroy` | Creates the replacement resource before destroying the old one | EC2 instances, load balancers — anything where downtime must be minimized during replacement |
| `prevent_destroy` | Causes `terraform destroy` (or any plan that would destroy this resource) to fail with an error | Production databases, S3 buckets with critical data — a safety net against accidental deletion |
| `ignore_changes` | Tells Terraform to ignore diffs in specific attributes after initial creation | Auto Scaling groups (where AWS manages instance count), resources modified by external processes (e.g. `ignore_changes = [tags]` when a tag is applied by a separate compliance tool) |

---

## Implicit vs Explicit Dependencies — In My Own Words

**Implicit dependencies** happen automatically when one resource references an attribute of another using the `resource_type.resource_name.attribute` syntax. Terraform reads these references, builds a directed acyclic graph (DAG), and determines the correct creation order without you specifying it. This covers ~95% of real-world cases.

**Explicit dependencies** via `depends_on` are needed when there's a **logical** dependency that Terraform can't see from attribute references. For example, if resource B needs resource A to exist, but B doesn't use any of A's output values — it just needs A's *side effects* (like an IAM policy being applied, a service being started, or a DNS record being propagated). In these cases you explicitly declare the ordering requirement.

The golden rule: **prefer implicit dependencies** (they're self-documenting). Use `depends_on` only when Terraform genuinely can't infer the relationship.

---

## Destroy Order

Running `terraform destroy` removes resources in **reverse dependency order** — the exact opposite of creation. The EC2 instance and S3 bucket are destroyed first, then the security group, then the route table association, route table, internet gateway, and subnet, and finally the VPC last. This ensures no resource is deleted while something that depends on it still exists.

---

## Key Takeaways

- The AWS provider is pinned with `~> 5.0` — safe minor/patch upgrades, no major version jumps
- `.terraform.lock.hcl` locks exact provider versions for reproducible runs across teams
- Terraform builds a DAG from implicit references — you define *what*, Terraform figures out *when*
- `depends_on` is the escape hatch for logical dependencies with no attribute reference
- `create_before_destroy` is essential for zero-downtime resource replacement in production
- Always run `terraform destroy` after practice to avoid unexpected AWS charges
