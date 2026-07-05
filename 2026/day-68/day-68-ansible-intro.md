# Day 68 - Introduction to Ansible

## Objective

Learn the fundamentals of Ansible by setting up a control node, creating an inventory, managing multiple EC2 instances over SSH, and executing ad-hoc commands.

---

# Task 1: Understanding Ansible

## What is Configuration Management?

Configuration Management is the process of maintaining servers in a consistent, automated, and repeatable state. Instead of configuring servers manually, configuration management tools automate software installation, service management, file configuration, and user management.

### Why do we need Configuration Management?

* Eliminates manual server configuration
* Maintains consistency across multiple servers
* Reduces human errors
* Saves time through automation
* Makes infrastructure reproducible
* Simplifies application deployment
* Supports Infrastructure as Code (IaC)

---

## How is Ansible different from Chef, Puppet, and Salt?

| Feature         | Ansible | Chef   | Puppet     | SaltStack   |
| --------------- | ------- | ------ | ---------- | ----------- |
| Agent Required  | No      | Yes    | Yes        | Optional    |
| Language        | YAML    | Ruby   | Puppet DSL | YAML/Python |
| Communication   | SSH     | Agent  | Agent      | SSH/Agent   |
| Push/Pull Model | Push    | Pull   | Pull       | Push/Pull   |
| Learning Curve  | Easy    | Medium | Medium     | Medium      |

### Key Difference

Ansible is **agentless**, making it simple to deploy and manage because no software needs to be installed on the managed servers.

---

## What does Agentless mean?

Agentless means no agent software is required on managed nodes.

Ansible communicates with Linux servers using **SSH** and executes modules remotely.

Requirements on managed nodes:

* SSH service
* Python installed

---

# Ansible Architecture

```
                    +---------------------------+
                    |       Control Node        |
                    |   Ubuntu EC2 + Ansible    |
                    +-------------+-------------+
                                  |
                           SSH (Port 22)
                                  |
        ---------------------------------------------------
        |                     |                          |
+----------------+    +----------------+      +----------------+
| Worker Ubuntu  |    | Worker Amazon  |      | Worker RedHat  |
| Managed Node   |    | Managed Node   |      | Managed Node   |
+----------------+    +----------------+      +----------------+

Inventory
    |
    +-- List of managed nodes

Modules
    |
    +-- ping
    +-- command
    +-- copy
    +-- apt
    +-- yum
    +-- shell
    +-- service

Playbooks
    |
    +-- YAML files defining automation tasks
```

---

# Ansible Components

## Control Node

The machine where Ansible is installed.

In my lab:

* Ubuntu EC2 Instance
* Executes all Ansible commands

---

## Managed Nodes

The servers managed by Ansible.

My lab consists of:

* Worker Ubuntu
* Worker Amazon Linux
* Worker Red Hat Enterprise Linux

---

## Inventory

Inventory contains information about all managed hosts.

```ini
[web]
worker-ubuntu ansible_host=<PUBLIC_IP> ansible_user=ubuntu

[app]
worker-amazon ansible_host=<PUBLIC_IP> ansible_user=ec2-user

[db]
worker-redhat ansible_host=<PUBLIC_IP> ansible_user=ec2-user

[application:children]
web
app

[all_servers:children]
application
db

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=/home/ubuntu/keys/terra-automate-key
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

---

## Modules

Modules are reusable units of work executed by Ansible.

Examples:

* ping
* command
* shell
* copy
* apt
* yum
* file
* service

---

## Playbooks

Playbooks are YAML files used to automate repeatable tasks.

Example:

```yaml
- hosts: web
  become: yes

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
```

---

# Task 2: Lab Setup

I provisioned my infrastructure using **Terraform**.

Resources created:

* AWS Default VPC
* Security Group
* SSH Key Pair
* Four EC2 Instances

| Instance            | Operating System  | Purpose              | Instance Type |
| ------------------- | ----------------- | -------------------- | ------------- |
| Control Node        | Ubuntu 22.04      | Ansible Control Node | t3.micro      |
| Worker Ubuntu       | Ubuntu 22.04      | Web Server           | t3.micro      |
| Worker Amazon Linux | Amazon Linux 2023 | Application Server   | t3.micro      |
| Worker Red Hat      | RHEL 9            | Database Server      | t3.micro      |

Security Group Rules:

* SSH (22)
* HTTP (80)

Verified SSH connectivity from the control node to every managed node.

---

# Task 3: Install Ansible

Installed Ansible on the Ubuntu Control Node.

```bash
sudo apt update
sudo apt install ansible -y
```

Verification:

```bash
ansible --version
```

The output displayed:

* Ansible Version
* Python Version
* Configuration File Location

### Why install Ansible only on the Control Node?

Ansible is agentless. Only the control node requires Ansible. Managed nodes only need:

* SSH
* Python

---

# Task 4: Inventory Configuration

Created the inventory file with separate host groups.

Verified connectivity:

```bash
ansible all -m ping
```

Output:

```
worker-ubuntu | SUCCESS => {"ping": "pong"}
worker-amazon | SUCCESS => {"ping": "pong"}
worker-redhat | SUCCESS => {"ping": "pong"}
```

---

# Task 5: Ad-Hoc Commands

## Check uptime

```bash
ansible all -m command -a "uptime"
```

## Check memory usage

```bash
ansible all -m command -a "free -h"
```

## Check disk usage

```bash
ansible all -m command -a "df -h"
```

## Install Git

Ubuntu:

```bash
ansible web -m apt -a "name=git state=present" --become
```

Amazon Linux / Red Hat:

```bash
ansible 'app:db' -m yum -a "name=git state=present" --become
```

## Copy a file

```bash
echo "Hello from Ansible" > hello.txt

ansible all -m copy -a "src=hello.txt dest=/tmp/hello.txt"
```

Verify:

```bash
ansible all -m command -a "cat /tmp/hello.txt"
```

---

## What does `--become` do?

`--become` executes commands with elevated (root) privileges using sudo.

It is required for:

* Installing packages
* Managing services
* Editing system configuration files
* Creating users
* Updating operating system packages

---

# Task 6: Inventory Groups and Patterns

### Run commands on Web servers

```bash
ansible web -m ping
```

### Run commands on Application servers

```bash
ansible app -m ping
```

### Run commands on Database servers

```bash
ansible db -m ping
```

### Run commands on Web + App

```bash
ansible application -m ping
```

### Run commands on all servers

```bash
ansible all_servers -m ping
```

### Inventory Patterns

Web OR App

```bash
ansible 'web:app' -m ping
```

All except Database

```bash
ansible 'all:!db' -m ping
```

---

# ansible.cfg

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ubuntu
private_key_file = /home/ubuntu/keys/terra-automate-key
```

This allows running commands without specifying the inventory file every time.

Example:

```bash
ansible all -m ping
```

---

# Difference Between Command and Shell Modules

| Command Module             | Shell Module                                  |
| -------------------------- | --------------------------------------------- |
| Executes commands directly | Executes commands through a shell             |
| More secure                | Supports pipes, redirects and shell variables |
| No shell interpretation    | Full shell functionality                      |
| Faster                     | Slightly slower                               |

Examples:

Command module:

```bash
ansible all -m command -a "uptime"
```

Shell module:

```bash
ansible all -m shell -a "cat /etc/passwd | grep ubuntu"
```

---

# Key Learnings

* Learned the fundamentals of Configuration Management.
* Understood Ansible's agentless architecture.
* Installed Ansible only on the control node.
* Created an inventory with multiple host groups.
* Connected to Ubuntu, Amazon Linux, and Red Hat EC2 instances over SSH.
* Executed ad-hoc commands using various Ansible modules.
* Configured inventory groups and host patterns.
* Used `--become` for privilege escalation.
* Compared the `command` and `shell` modules.
* Managed multiple AWS EC2 instances from a single terminal without installing any agents.

---

# Conclusion

Today I started my Ansible journey by setting up a dedicated control node, provisioning AWS infrastructure with Terraform, creating an inventory of managed EC2 instances, and successfully running Ansible ad-hoc commands across multiple servers. This hands-on lab demonstrated how Ansible simplifies server management through SSH-based, agentless automation.
