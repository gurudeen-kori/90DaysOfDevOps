# Day 79 – Creating a Custom Helm Chart for AI-BankApp

## Overview

On Day 79, the AI-BankApp was migrated from **raw Kubernetes manifests** to a **custom Helm Chart**. The objective was to make the application configurable, reusable, and easier to manage across different environments.

The original application consisted of 12 Kubernetes YAML manifests for deploying:

- Spring Boot AI-BankApp
- MySQL Database
- Ollama AI Chatbot
- ConfigMaps
- Secrets
- Persistent Volumes
- Services
- Horizontal Pod Autoscaler
- Gateway & TLS Resources

Using Helm, all these resources can now be deployed with a single command:

```bash
helm install my-bankapp ./bankapp
```

---

# Raw Kubernetes Manifests vs Helm Templates

## 1. Deployment

### Raw Manifest

**File:** `k8s/bankapp-deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bankapp

spec:
  replicas: 4

template:
  spec:
    containers:
      - name: bankapp
        image: trainwithshubham/ai-bankapp-eks:latest
```

### Helm Template

**File:** `templates/bankapp-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: {{ include "bankapp.fullname" . }}

spec:
  replicas: {{ .Values.bankapp.replicaCount }}

template:
  spec:
    containers:
      - name: bankapp
        image: "{{ .Values.bankapp.image.repository }}:{{ .Values.bankapp.image.tag }}"
```

### Improvements

- Replica count is configurable.
- Image repository and tag are configurable.
- Reusable across environments.
- No hardcoded values.

---

## 2. Service

### Raw Manifest

```yaml
apiVersion: v1
kind: Service

metadata:
  name: bankapp-service

spec:
  type: ClusterIP

ports:
- port: 8080
```

### Helm Template

```yaml
apiVersion: v1
kind: Service

metadata:
  name: {{ include "bankapp.fullname" . }}-service

spec:
  type: {{ .Values.bankapp.service.type }}

ports:
- port: {{ .Values.bankapp.service.port }}
```

### Improvements

- Service type configurable.
- Port configurable.
- Dynamic naming.

---

## 3. Secret

### Raw Manifest

```yaml
data:
  MYSQL_PASSWORD: VGVzdEAxMjM=
```

### Helm Template

```yaml
data:
  MYSQL_PASSWORD: {{ .Values.secrets.mysqlPassword | b64enc | quote }}
```

### Improvements

- No manual Base64 encoding.
- Credentials managed through `values.yaml`.
- Environment-specific secrets.

---

# Complete values.yaml Explained

| Section | Description |
|----------|-------------|
| bankapp | Spring Boot application configuration |
| mysql | MySQL image, storage, resources |
| ollama | AI model configuration |
| config | Shared ConfigMap values |
| secrets | Database credentials |
| storageClass | Persistent storage configuration |
| gateway | Optional Envoy Gateway configuration |

Example:

```yaml
bankapp:
  replicaCount: 4

  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: latest

mysql:
  persistence:
    size: 5Gi

ollama:
  model: tinyllama

config:
  mysqlDatabase: bankappdb

secrets:
  mysqlRootPassword: Test@123
```

---

# Helm Go Template Cheat Sheet

| Template | Purpose |
|----------|---------|
| `{{ .Values }}` | Access values from values.yaml |
| `{{ if }}` | Conditional rendering |
| `{{ range }}` | Loop through lists/maps |
| `{{ with }}` | Simplify nested values |
| `{{ include }}` | Include helper template |
| `toYaml` | Convert object to YAML |
| `nindent` | Indent YAML correctly |
| `b64enc` | Base64 encode strings |

Examples:

```yaml
{{ .Values.bankapp.replicaCount }}
```

```yaml
{{ if .Values.mysql.enabled }}
...
{{ end }}
```

```yaml
{{ include "bankapp.fullname" . }}
```

```yaml
{{- toYaml .Values.resources | nindent 12 }}
```

```yaml
{{ .Values.secrets.mysqlPassword | b64enc }}
```

---

# Rendering Templates

Validate the chart:

```bash
helm lint bankapp/
```

Render manifests locally:

```bash
helm template my-bankapp bankapp/
```

Example output:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: my-bankapp

spec:
  replicas: 4
```

This command generates the final Kubernetes manifests without deploying them.

---

# Deploying on Kind

Install:

```bash
helm install my-bankapp bankapp \
-n bankapp \
--create-namespace \
--set storageClass.create=false \
--set mysql.persistence.storageClass=standard \
--set ollama.persistence.storageClass=standard
```

Verify deployment:

```bash
helm list -n bankapp

kubectl get all -n bankapp

kubectl get pvc -n bankapp

kubectl get configmap,secret -n bankapp
```

Port forward:

```bash
kubectl port-forward svc/my-bankapp-bankapp-service \
-n bankapp 8080:8080
```

Open:

```
http://localhost:8080
```

---

# Disabling Ollama

One of the biggest advantages of Helm templates is conditional resource creation.

Disable Ollama:

```bash
helm install my-bankapp bankapp \
--set ollama.enabled=false
```

Resources automatically removed:

- Ollama Deployment
- Ollama Service
- Ollama PVC
- Ollama Init Container

No template modifications are required.

---

# Benefits of Helm

- Reusable templates
- Environment-specific configuration
- Single deployment command
- Easier upgrades
- Rollback support
- Version-controlled releases
- No duplicate YAML files
- Better maintainability

---

# Conclusion

Migrating from raw Kubernetes manifests to a Helm Chart significantly improved the deployment process for AI-BankApp. Instead of maintaining multiple static YAML files, Helm provides a centralized, configurable, and reusable deployment model using templates and `values.yaml`.

The chart was successfully validated using:

```bash
helm lint
```

and rendered using:

```bash
helm template
```

making it production-ready for deployment on Kubernetes clusters.