# Ansible Docker & Nginx Automation Project

## Project Overview

This project demonstrates Infrastructure as Code (IaC) and Configuration Management by automating the deployment of a Dockerized application using Ansible. The automation provisions Linux servers, installs Docker, deploys an application container, configures Nginx as a reverse proxy, and secures Docker Hub credentials using Ansible Vault.

The solution is modular, reusable, idempotent, and follows Ansible best practices by organizing automation into roles.

---

## Objectives

* Automate Linux server configuration
* Install and configure Docker Engine
* Deploy Docker containers automatically
* Configure Nginx as a Reverse Proxy
* Secure Docker Hub credentials using Ansible Vault
* Implement reusable Ansible Roles
* Support idempotent deployments
* Reduce manual server administration

---

## Technology Stack

| Category                 | Technologies                             |
| ------------------------ | ---------------------------------------- |
| Configuration Management | Ansible                                  |
| Containerization         | Docker, Docker Hub                       |
| Web Server               | Nginx                                    |
| Operating Systems        | Red Hat Enterprise Linux 9, Amazon Linux |
| Cloud Platform           | AWS EC2                                  |
| Templating               | Jinja2                                   |
| Scripting                | YAML                                     |
| Secret Management        | Ansible Vault                            |
| Version Control          | Git, GitHub                              |

---

## Project Structure

```text
ansible-docker-project/
│
├── ansible.cfg
├── hosts.ini
├── site.yml
│
├── group_vars/
│   ├── all.yml
│   └── web/
│       └── vault.yml
│
├── roles/
│   ├── common/
│   ├── docker/
│   └── nginx/
│
└── terra-automate-key
```

---

# Role Overview

## 1. Common Role

The Common role prepares every managed server before application deployment.

### Responsibilities

* Update package cache
* Install common Linux utilities
* Configure hostname
* Configure timezone
* Create deployment user
* Maintain consistent baseline configuration

### Common Packages

* vim
* curl
* wget
* git
* jq
* unzip
* tree
* htop

---

## 2. Docker Role

The Docker role automates Docker installation and application deployment.

### Responsibilities

* Install Docker dependencies
* Configure Docker CE repository
* Install Docker Engine
* Enable Docker service
* Add deploy user to docker group
* Install Python Docker SDK
* Authenticate with Docker Hub
* Pull Docker image
* Deploy Docker container

### Docker Variables

```yaml
docker_app_image: nginx
docker_app_tag: latest
docker_app_name: myapp
docker_app_port: 8080
docker_container_port: 80
```

Changing the application image only requires updating variables without modifying playbooks.

---

## 3. Nginx Role

The Nginx role exposes the Docker application through a reverse proxy.

### Responsibilities

* Install Nginx
* Remove default configuration
* Deploy nginx.conf using Jinja2 templates
* Deploy reverse proxy configuration
* Validate Nginx configuration
* Enable and start Nginx service
* Reload Nginx using handlers when configuration changes

---

# Configuration Management

The project follows Ansible best practices by separating:

* Roles
* Variables
* Templates
* Handlers
* Defaults
* Inventory
* Vault Secrets

This modular design improves maintainability and scalability.

---

# Security

Sensitive information is protected using Ansible Vault.

Encrypted credentials include:

* Docker Hub Username
* Docker Hub Password

Vault encryption prevents secrets from being stored in plain text within the repository.

---

# Deployment Workflow

1. Configure inventory
2. Configure group variables
3. Encrypt credentials using Ansible Vault
4. Execute playbook
5. Install common packages
6. Install Docker
7. Authenticate with Docker Hub
8. Pull application image
9. Start Docker container
10. Configure Nginx reverse proxy
11. Validate Nginx configuration
12. Reload Nginx if configuration changes

---

# Running the Project

## Dry Run

```bash
ansible-playbook site.yml --check --diff
```

## Full Deployment

```bash
ansible-playbook site.yml
```

---

# Tag-Based Execution

Run only Common Role

```bash
ansible-playbook site.yml --tags common
```

Run only Docker Role

```bash
ansible-playbook site.yml --tags docker
```

Run only Nginx Role

```bash
ansible-playbook site.yml --tags nginx
```

Skip Common Role

```bash
ansible-playbook site.yml --skip-tags common
```

---

# Verification

Verify Docker

```bash
docker ps
```

Verify Images

```bash
docker images
```

Verify Docker Service

```bash
systemctl status docker
```

Verify Nginx Service

```bash
systemctl status nginx
```

Verify Reverse Proxy

```bash
curl http://<server-ip>
```

---

# Challenges Encountered

## Docker SDK Missing

**Issue**

```
No module named docker
```

**Resolution**

* Installed python3-pip
* Installed Docker SDK using pip

---

## Invalid Hostname

**Issue**

Hostnames containing underscores were rejected.

**Resolution**

Used Linux-compatible hostnames with hyphens.

---

## Docker Image Not Found

**Issue**

Incorrect Docker image name or tag.

**Resolution**

Updated the image repository and tag to match Docker Hub.

---

## Out of Memory (OOM)

**Issue**

Python process was terminated during package installation on a low-memory EC2 instance.

**Resolution**

Used an instance with sufficient memory and re-ran the playbook.

---

# Key Features

* Modular Ansible Roles
* Idempotent Playbooks
* Docker Automation
* Nginx Reverse Proxy
* Jinja2 Templates
* Secret Management with Ansible Vault
* Role-Based Project Structure
* Tag-Based Deployment
* AWS EC2 Deployment
* Production-Oriented Configuration

---

# Skills Demonstrated

* Linux Administration
* Ansible Playbooks
* Ansible Roles
* Infrastructure as Code (IaC)
* Configuration Management
* Docker
* Docker Hub
* Nginx
* Reverse Proxy Configuration
* YAML
* Jinja2 Templates
* Secret Management
* AWS EC2
* SSH Automation
* Troubleshooting
* Idempotent Automation

---

# Learning Outcomes

This project provided practical experience with:

* Automating server provisioning
* Managing infrastructure using Ansible
* Deploying containerized applications
* Configuring reverse proxies
* Implementing secure secret management
* Creating reusable automation components
* Applying Infrastructure as Code principles

---

# Future Enhancements

* HTTPS with Let's Encrypt
* Docker Compose Support
* Multi-Container Applications
* Kubernetes Deployment
* Prometheus Monitoring
* Grafana Dashboards
* CI/CD Pipeline using GitHub Actions
* Blue-Green Deployment
* Rolling Updates
* Auto Scaling Support

---

# Conclusion

This project demonstrates a production-oriented approach to infrastructure automation using Ansible. By combining reusable roles, secure secret management, Docker container deployment, and Nginx reverse proxy configuration, the solution enables consistent, repeatable, and scalable deployments across Linux servers.

The automation significantly reduces manual effort, improves deployment consistency, and follows industry best practices for DevOps and Infrastructure as Code.
