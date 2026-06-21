# Day 59 – Helm: Kubernetes Package Manager

## Objective

Learn how to use Helm to package, deploy, customize, upgrade, rollback, and manage Kubernetes applications efficiently.

---

# What is Helm?

Helm is the package manager for Kubernetes, similar to how apt works for Ubuntu or yum works for CentOS.

Instead of manually managing multiple YAML files, Helm packages Kubernetes manifests into reusable units called Charts.

Benefits:

* Simplifies Kubernetes deployments
* Supports configuration through values files
* Provides versioning and rollback capabilities
* Enables reusable application packaging
* Reduces YAML duplication

---

# Three Core Concepts

## 1. Chart

A Chart is a package containing Kubernetes manifest templates.

Example:

```bash
helm install my-nginx bitnami/nginx
```

Here:

```text
bitnami/nginx
```

is the chart.

---

## 2. Release

A Release is a running instance of a chart inside a Kubernetes cluster.

Example:

```bash
helm install my-nginx bitnami/nginx
```

Here:

```text
my-nginx
```

is the release name.

---

## 3. Repository

A Repository is a collection of Helm charts.

Example:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Bitnami hosts hundreds of production-ready charts.

---

# Task 1 – Install Helm

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Verify

```bash
helm version
helm env
```

Example Output:

```text
version.BuildInfo{Version:"v3.x.x"}
```

---

# Task 2 – Add Repository and Search Charts

## Add Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

## Update Repository

```bash
helm repo update
```

## Search Charts

```bash
helm search repo nginx
helm search repo bitnami
```

Bitnami provides hundreds of Helm charts for Kubernetes workloads.

---

# Task 3 – Install a Chart

## Deploy Nginx

```bash
helm install my-nginx bitnami/nginx
```

## Verify

```bash
helm list

helm status my-nginx

kubectl get all
```

## Inspect Rendered Manifests

```bash
helm get manifest my-nginx
```

Helm automatically created:

* Deployment
* Service
* ConfigMap
* Secret
* ServiceAccount

without manually writing YAML files.

---

# Task 4 – Customize with Values

## View Default Values

```bash
helm show values bitnami/nginx
```

## Install Using --set

```bash
helm install my-nginx-server bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort
```

---

## custom-values.yaml

```yaml
replicaCount: 3

service:
  type: NodePort

resources:
  requests:
    cpu: 100m
    memory: 128Mi

  limits:
    cpu: 200m
    memory: 256Mi
```

---

## Install Using Values File

```bash
helm install my-nginx-values bitnami/nginx \
  -f custom-values.yaml
```

## Check Applied Values

```bash
helm get values my-nginx-values
```

## Verify

```bash
kubectl get deployment
kubectl get svc
```

Expected:

```text
Replicas: 3
Service Type: NodePort
```

---

# Task 5 – Upgrade and Rollback

## Upgrade Release

```bash
helm upgrade my-nginx bitnami/nginx \
  --set replicaCount=5
```

## Check History

```bash
helm history my-nginx
```

Example:

```text
REVISION STATUS
1        deployed
2        deployed
```

---

## Rollback

```bash
helm rollback my-nginx 1
```

Check history again:

```bash
helm history my-nginx
```

Example:

```text
REVISION STATUS
1        deployed
2        superseded
3        deployed
```

Rollback creates a new revision instead of overwriting previous revisions.

---

# Task 6 – Create Your Own Chart

## Create Chart

```bash
helm create my-app
```

Generated Structure:

```text
my-app/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ...
└── charts/
```

---

# Important Files

## Chart.yaml

Contains chart metadata.

```yaml
name: my-app
version: 0.1.0
```

---

## values.yaml

Stores configurable parameters.

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
```

---

## templates/deployment.yaml

Uses Go Templates.

Example:

```yaml
replicas: {{ .Values.replicaCount }}
```

Helm replaces placeholders during rendering.

Example:

```yaml
replicas: 3
```

---

# Go Templating Examples

## Values

```yaml
{{ .Values.replicaCount }}
```

Reads values.yaml.

---

## Chart Name

```yaml
{{ .Chart.Name }}
```

Returns chart name.

---

## Release Name

```yaml
{{ .Release.Name }}
```

Returns Helm release name.

---

# Validate Chart

```bash
helm lint my-app
```

Checks:

* YAML syntax
* Template validity
* Chart structure

---

# Render Without Installing

```bash
helm template my-release ./my-app
```

Useful for debugging.

---

# Install Chart

```bash
helm install github-app ./my-app -n github-app
```

Verify:

```bash
kubectl get all -n github-app
```

Result:

```text
Deployment: 3 replicas
Service: NodePort
Pods: Running
```

---

# Upgrade Chart

```bash
helm upgrade github-app ./my-app \
  -n github-app \
  --set replicaCount=5
```

Verify:

```bash
kubectl get deployment -n github-app
```

Expected:

```text
5 replicas running
```

---

# Task 7 – Clean Up

## Uninstall Releases

```bash
helm uninstall my-nginx

helm uninstall my-nginx-server

helm uninstall github-app -n github-app

helm uninstall github-app -n github
```

## Verify

```bash
helm list -A
```

Expected:

```text
No releases found
```

---

# Key Learnings

* Helm is the package manager for Kubernetes.
* Charts package Kubernetes manifests.
* Releases are installed chart instances.
* Repositories store collections of charts.
* Values files provide customization.
* helm upgrade updates existing releases.
* helm rollback restores previous versions.
* helm template renders manifests without installation.
* helm lint validates chart quality.
* Go templates make charts reusable and dynamic.

---

# Conclusion

Today I learned Helm fundamentals by deploying Bitnami charts, customizing installations with values files, performing upgrades and rollbacks, and building a custom chart from scratch.

One Helm command can replace dozens of manually maintained Kubernetes YAML files, making application deployment faster, repeatable, and easier to manage.
