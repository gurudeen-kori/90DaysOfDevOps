# Day 61 - Terraform Introduction

## Task 1: Understanding Infrastructure as Code (IaC)

### What is Infrastructure as Code (IaC)?

Infrastructure as Code (IaC) is the practice of creating and managing infrastructure through code instead of manually creating resources in the cloud console. It allows infrastructure to be version-controlled, automated, and reproducible.

### Why does IaC matter in DevOps?

IaC eliminates manual configuration errors, improves consistency, enables automation, and allows teams to manage infrastructure using the same workflows used for application code.

### What problems does IaC solve?

When infrastructure is created manually, mistakes and configuration drift can occur. IaC provides a single source of truth and ensures that environments can be recreated reliably whenever needed.

### Terraform vs CloudFormation vs Ansible vs Pulumi

* Terraform is cloud-agnostic and works across AWS, Azure, GCP, and many other providers.
* CloudFormation is AWS-specific.
* Ansible focuses mainly on configuration management and automation.
* Pulumi uses programming languages like Python, JavaScript, and Go instead of HCL.

### Declarative and Cloud-Agnostic

Terraform is declarative because we define the desired end state and Terraform determines how to achieve it.

Terraform is cloud-agnostic because the same workflow can be used across multiple cloud providers.

---

## Task 2: Install Terraform and Configure AWS

### Verify Terraform Installation

```bash
terraform -version
```

### Configure AWS CLI

```bash
aws configure
```

### Verify AWS Access

```bash
aws sts get-caller-identity
```

This command confirms that Terraform and AWS CLI can authenticate successfully with AWS.

---

## Task 3: Create Infrastructure Using Terraform

### Resources Created

1. AWS Key Pair
2. Default VPC
3. Security Group
4. HTTP Ingress Rule (Port 80)
5. SSH Ingress Rule (Port 22)
6. Outbound Traffic Rule
7. Five EC2 Instances

### Terraform Configuration Overview

#### Key Pair

Creates an SSH key pair used for connecting to EC2 instances.

#### Default VPC

Uses the existing default VPC in the AWS account.

#### Security Group

Creates a security group allowing:

* SSH (22)
* HTTP (80)

#### EC2 Instances

Terraform creates five EC2 instances using:

* AMI: ami-01a00762f46d584a1
* Instance Type: t3.micro
* Root Volume: 10 GB gp3
* Tags: terra-auto-server

### Terraform Lifecycle

```bash
terraform init
terraform validate
terraform fmt
terraform plan
terraform apply
```

### What did terraform init download?

Terraform downloaded the AWS provider plugin and initialized the project directory.

### What does the .terraform directory contain?

* AWS provider plugins
* Dependency information
* Internal Terraform metadata
* Downloaded modules

---

## Task 4: Verify Infrastructure

After running terraform apply:

### Resources Created Successfully

* 1 Key Pair
* 1 Security Group
* 2 Security Group Rules
* 5 EC2 Instances

Verified from:

* AWS EC2 Console
* AWS VPC Console
* AWS Security Group Console

---

## Task 5: Understanding Terraform State

### terraform show

Displays a human-readable representation of the current infrastructure state.

### terraform state list

Lists all resources managed by Terraform.

Example:

```bash
terraform state list
```

Output includes:

```text
aws_instance.my_instance[0]
aws_instance.my_instance[1]
aws_instance.my_instance[2]
aws_instance.my_instance[3]
aws_instance.my_instance[4]
aws_key_pair.my_key
aws_security_group.my_security_group
```

### terraform state show

Displays detailed information about a specific resource.

Example:

```bash
terraform state show aws_instance.my_instance[0]
```

### What information does terraform.tfstate store?

* Resource IDs
* Public and Private IPs
* Instance metadata
* Security Group IDs
* Key Pair details
* Resource dependencies

### Why should you never manually edit the state file?

Manual changes can corrupt the state and cause Terraform to lose track of resources.

### Why should state files not be committed to Git?

State files may contain sensitive information such as resource identifiers, IP addresses, and infrastructure metadata.

---

## Task 6: Modify, Plan and Destroy

### Modification

Changed EC2 tag:

Before:

```hcl
tags = {
  Name = "terra-auto-server"
}
```

After:

```hcl
tags = {
  Name = "terra-auto-server-modified"
}
```

### Terraform Plan Symbols

| Symbol | Meaning                  |
| ------ | ------------------------ |
| +      | Create Resource          |
| -      | Destroy Resource         |
| ~      | Modify Existing Resource |

### Was it an in-place update?

Yes. Only the tag changed, so Terraform updated the resource without recreating it.

### Apply Changes

```bash
terraform apply
```

### Destroy Infrastructure

```bash
terraform destroy
```

This removed:

* EC2 Instances
* Security Group Rules
* Security Group
* Key Pair

---

## Commands Used

| Command              | Purpose                |
| -------------------- | ---------------------- |
| terraform init       | Initialize project     |
| terraform validate   | Validate syntax        |
| terraform fmt        | Format code            |
| terraform plan       | Preview changes        |
| terraform apply      | Create infrastructure  |
| terraform show       | Show state             |
| terraform state list | List managed resources |
| terraform state show | Show resource details  |
| terraform destroy    | Remove infrastructure  |

---

## Key Learnings

* Infrastructure can be fully managed through code.
* Terraform tracks resources using a state file.
* Security Groups can be managed independently using dedicated ingress and egress resources.
* Multiple EC2 instances can be created using the count meta-argument.
* Terraform makes infrastructure reproducible, version-controlled, and easy to automate.
