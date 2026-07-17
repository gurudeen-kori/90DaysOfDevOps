# Day 78: Introduction to Helm and Chart Basics

## Overview
Today I learned about Helm, the package manager for Kubernetes. Instead of managing multiple YAML files, Helm provides templating, versioning, and dependency management for Kubernetes applications.

---

## Task 1: Helm Concepts

### What is Helm?

**Helm** is the package manager for Kubernetes, similar to:
- `apt` for Ubuntu/Debian
- `yum` for RHEL/CentOS
- `brew` for macOS
- `npm` for Node.js

Helm packages Kubernetes manifests into reusable, versioned units called **charts**. It enables:
- **Templating**: One chart works across multiple environments (dev, staging, prod)
- **Versioning**: Track and rollback to previous versions
- **Dependency Management**: Charts can depend on other charts
- **Reusability**: Share charts across teams and projects

### Core Helm Concepts

#### 1. **Chart**
A chart is a collection of YAML files that describe a complete Kubernetes application.

**Example**: A MySQL chart includes:
- StatefulSet (for MySQL pod)
- Service (for network access)
- Secret (for credentials)
- ConfigMap (for configuration)
- PersistentVolumeClaim (for storage)
- ServiceMonitor (for metrics)

Instead of managing 5+ separate YAML files, one chart handles everything.

#### 2. **Release**
A release is a running instance of a chart in your cluster.

You can install the same chart multiple times with different release names:
```bash
helm install mysql-prod bitnami/mysql --set auth.rootPassword=prod-pass
helm install mysql-dev bitnami/mysql --set auth.rootPassword=dev-pass
```
Now you have two MySQL releases with different configurations.

#### 3. **Repository**
A repository is a remote storage for Helm charts (like DockerHub for container images).

Official repositories:
- **Bitnami**: https://charts.bitnami.com/bitnami (most popular)
- **Stable** (deprecated): https://charts.helm.sh/stable
- **ArtifactHub**: https://artifacthub.io (search all charts)

#### 4. **Values**
Values are configuration parameters that customize a chart for each deployment.

They can be:
- **Passed via CLI**: `--set key=value`
- **Stored in files**: `-f values.yaml`
- **Merged together**: CLI overrides file values

Example values for MySQL:
```yaml
auth:
  rootPassword: Test@123
  database: bankappdb
primary:
  replicaCount: 1
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
```

### Why Helm Over Raw Manifests?

#### AI-BankApp's Problem (12 Raw YAML Files)

```
k8s/
├── namespace.yml              # 1. Define namespace
├── secrets.yml                # 2. Database credentials (hardcoded base64)
├── configmap.yml              # 3. App configuration
├── pv.yml                      # 4. PersistentVolume
├── pvc.yml                     # 5. PersistentVolumeClaim
├── mysql-deployment.yml        # 6. MySQL StatefulSet
├── service.yml                 # 7. MySQL Service
├── bankapp-deployment.yml      # 8. App Deployment
├── ollama-deployment.yml       # 9. Ollama Deployment
├── gateway.yml                 # 10. API Gateway
├── hpa.yml                     # 11. Horizontal Pod Autoscaler
└── cert-manager.yml            # 12. Certificate Manager
```

**Problems:**
- All values are hardcoded
- To change image tag → edit 3 files
- To switch to production → manually update all secrets
- No rollback capability → manual git revert
- Scaling → edit replicas in YAML files
- No versioning → hard to track what changed

**Helm Solutions:**
- One chart, different values per environment
- `helm upgrade` tracks changes automatically
- `helm rollback` reverts to previous versions instantly
- `--set` overrides values on the fly
- Charts can have versions (v1.0.0, v1.1.0, etc.)

---

## Task 3 & 4: Deploying MySQL with Helm

### Installing Bitnami MySQL Chart

#### Add Repository
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

#### Search for MySQL Chart
```bash
helm search repo bitnami/mysql
```

#### Deploy with CLI Arguments
```bash
helm install bankapp-mysql bitnami/mysql \
  --set auth.rootPassword=Test@123 \
  --set auth.database=bankappdb \
  --set primary.resources.requests.memory=256Mi \
  --set primary.resources.requests.cpu=250m \
  --set primary.resources.limits.memory=512Mi \
  --set primary.resources.limits.cpu=500m \
  --set primary.persistence.size=5Gi
```

**What this single command creates:**
- StatefulSet with MySQL pod
- Service for network access
- Secret for root password
- ConfigMap for my.cnf configuration
- PersistentVolumeClaim for data storage
- All labeled and managed by Helm

#### Compare: Raw YAML Approach

To achieve the same with raw YAML, you would need:

**mysql-deployment.yml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: bitnami/mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: bankappdb
        resources:
          requests:
            memory: 256Mi
            cpu: 250m
          limits:
            memory: 512Mi
            cpu: 500m
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

**secrets.yml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  password: VGVzdEAxMjM=  # Base64 encoded
```

**service.yml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  clusterIP: None
  selector:
    app: mysql
```

**pvc.yml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

**The Helm Advantage:**
- One command vs 4+ files
- No manual base64 encoding
- Automatic label management
- Built-in validation
- Easy to version and rollback

### Using Values File (Recommended Approach)

#### Create `mysql-values.yaml`

```yaml
# MySQL Authentication
auth:
  rootPassword: Test@123          # Root user password
  database: bankappdb             # Database to create automatically
  username: ""                    # Additional user (optional)
  password: ""                    # Additional user password

# Primary MySQL Pod Configuration
primary:
  # Number of MySQL pods (replicas)
  replicaCount: 1
  
  # Resource allocation
  resources:
    limits:
      cpu: 500m                   # Maximum CPU
      memory: 512Mi               # Maximum memory
    requests:
      cpu: 250m                   # Minimum CPU
      memory: 256Mi               # Minimum memory
  
  # Persistent storage configuration
  persistence:
    size: 5Gi                     # Storage volume size
    storageClass: ""              # Use default storage class
    accessMode: ReadWriteOnce

# Metrics collection (for monitoring)
metrics:
  enabled: true                   # Enable Prometheus metrics
  serviceMonitor:
    enabled: false                # Don't create ServiceMonitor CRD

# Image configuration
image:
  repository: bitnami/mysql       # Docker image repository
  tag: 8.0                        # Image tag

# Port configuration
primary:
  service:
    port: 3306                    # MySQL port
```

#### Explanation of Each Field

| Field | Purpose | Example |
|-------|---------|---------|
| `auth.rootPassword` | MySQL root user password | `Test@123` |
| `auth.database` | Initial database to create | `bankappdb` |
| `primary.replicaCount` | Number of MySQL replicas | `1` (no replication) |
| `primary.resources.requests.memory` | Guaranteed minimum memory | `256Mi` |
| `primary.resources.requests.cpu` | Guaranteed minimum CPU | `250m` |
| `primary.resources.limits.memory` | Maximum allowed memory | `512Mi` |
| `primary.resources.limits.cpu` | Maximum allowed CPU | `500m` |
| `primary.persistence.size` | Persistent volume size | `5Gi` |
| `metrics.enabled` | Enable Prometheus metrics | `true` |

#### Deploy with Values File

```bash
helm install bankapp-mysql bitnami/mysql -f mysql-values.yaml
```

#### View All Available Values

```bash
helm show values bitnami/mysql | head -100
```

This shows all configurable options available in the chart.

---

## Task 5: Release Management

### Upgrade Releases

Enable metrics monitoring:
```bash
helm upgrade bankapp-mysql bitnami/mysql -f mysql-values.yaml \
  --set metrics.enabled=true
```

### Check Release History

```bash
helm history bankapp-mysql
```

Output:
```
REVISION	UPDATED                 	STATUS    	CHART           	APP VERSION	DESCRIPTION     
1       	Thu Jul 17 19:59:05 2026	deployed  	mysql-12.2.1    	8.0.40     	Install complete
2       	Thu Jul 17 20:15:30 2026	deployed  	mysql-12.2.1    	8.0.40     	Upgrade complete
```

**Revision tracking benefits:**
- Each change is recorded
- Can see what changed and when
- Easy to rollback if something breaks

### Rollback to Previous Version

```bash
helm rollback bankapp-mysql 1
```

Output:
```
Rollback "bankapp-mysql" from revision 2 to 1
```

Check history again:
```bash
helm history bankapp-mysql
```

```
REVISION	UPDATED                 	STATUS    	CHART           	APP VERSION	DESCRIPTION     
1       	Thu Jul 17 19:59:05 2026	deployed  	mysql-12.2.1    	8.0.40     	Install complete
2       	Thu Jul 17 20:15:30 2026	superseded	mysql-12.2.1    	8.0.40     	Upgrade complete
3       	Thu Jul 17 20:20:45 2026	deployed  	mysql-12.2.1    	8.0.40     	Rollback to 1
```

**Note:** Rollback creates a new revision (3), it doesn't delete revision 2. This ensures full audit trail.

### Uninstall Release

```bash
helm uninstall bankapp-mysql
```

This removes:
- StatefulSet
- Service
- ConfigMaps
- Secrets
- PersistentVolumeClaims (optional)

---

## Task 6: Chart Structure

### Explore Helm Chart Directory

```bash
helm pull bitnami/mysql --untar
ls -la mysql/
```

### Chart Directory Structure

```
mysql/
├── Chart.yaml                      # Chart metadata
├── values.yaml                     # Default configuration values
├── values-production.yaml          # Production-specific values
├── README.md                        # Chart documentation
├── charts/                          # Subchart dependencies
│   └── common/                      # Common helper functions
├── templates/                       # Kubernetes manifest templates
│   ├── NOTES.txt                   # Post-install message
│   ├── _helpers.tpl                # Reusable template functions
│   ├── secrets.yaml                # Secret template
│   ├── configmap.yaml              # ConfigMap template
│   ├── primary/                    # Primary MySQL templates
│   │   ├── statefulset.yaml        # StatefulSet template
│   │   └── svc.yaml                # Service template
│   ├── secondary/                  # Secondary MySQL templates (for replication)
│   │   ├── statefulset.yaml
│   │   └── svc.yaml
│   ├── metrics/                    # Metrics exporter templates
│   │   └── servicemonitor.yaml
│   └── tests/                      # Test templates
└── .helmignore                     # Files to ignore in chart
```

### Chart Metadata: Chart.yaml

```yaml
apiVersion: v2                      # Helm 3 (v2) or Helm 2 (v1)
name: mysql                         # Chart name
description: A Helm chart for MySQL # Chart description
type: application                   # Chart type (application or library)

# Version of the chart itself (when you make changes to the chart)
version: 12.2.1

# Version of the MySQL application inside the chart
appVersion: "8.0.40"

# Chart maintainers
maintainers:
  - name: Bitnami
    email: containers@bitnami.com

# Keywords for searching
keywords:
  - mysql
  - database

# Chart dependencies
dependencies:
  - name: common
    version: "1.x.x"
    repository: "https://charts.bitnami.com/bitnami"

# License
license: Apache-2.0
```

#### Difference: `version` vs `appVersion`

| Aspect | version | appVersion |
|--------|---------|-----------|
| **Meaning** | Chart version (chart structure/features) | Application version (MySQL version) |
| **When it changes** | When you modify the chart | When MySQL version is updated |
| **Example v1** | mysql-12.0.0 | MySQL 8.0.30 |
| **Example v2** | mysql-12.1.0 | MySQL 8.0.30 (but chart added metrics) |
| **Example v3** | mysql-12.2.1 | MySQL 8.0.40 (both chart and MySQL updated) |

**Use Case:**
```bash
# Install specific chart version with specific MySQL version
helm install mysql bitnami/mysql --version 12.2.1
# Will get MySQL 8.0.40 from this chart version

# Upgrade chart but keep MySQL version
helm upgrade mysql bitnami/mysql --version 12.1.5
# Might get MySQL 8.0.30 if chart 12.1.5 had that version
```

### Template Example: statefulset.yaml

Raw template with Go syntax:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "mysql.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  serviceName: {{ include "mysql.fullname" . }}
  replicas: {{ .Values.primary.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "mysql.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "mysql.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      containers:
      - name: mysql
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: mysql
          containerPort: {{ .Values.primary.service.port }}
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "mysql.fullname" . }}
              key: mysql-root-password
        - name: MYSQL_DATABASE
          value: {{ .Values.auth.database }}
        resources:
          limits:
            cpu: {{ .Values.primary.resources.limits.cpu }}
            memory: {{ .Values.primary.resources.limits.memory }}
          requests:
            cpu: {{ .Values.primary.resources.requests.cpu }}
            memory: {{ .Values.primary.resources.requests.memory }}
        volumeMounts:
        - name: data
          mountPath: /bitnami/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: {{ .Values.primary.persistence.size }}
```

**Go Template Syntax:**
- `{{ .Values.primary.replicaCount }}` → Inserts value from values.yaml
- `{{ include "mysql.fullname" . }}` → Calls a helper function
- `{{ .Release.Name }}` → Inserts release name
- `{{ .Release.Namespace }}` → Inserts namespace

**Example Rendering:**

With values.yaml containing `primary.replicaCount: 1`, the template above renders to:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: bankapp-mysql
  namespace: default
spec:
  serviceName: bankapp-mysql
  replicas: 1  # ← Inserted from values
  selector:
    matchLabels:
      app.kubernetes.io/name: mysql
      app.kubernetes.io/instance: bankapp-mysql
  # ... rest of manifest
```

---

## Comparison: Raw YAML vs Helm Chart

### AI-BankApp MySQL Example

| Aspect | Raw YAML Files | Bitnami Helm Chart |
|--------|---|---|
| **Files to manage** | 5+ (deployment, service, secret, pvc, pv, configmap) | 1 chart |
| **Secrets** | Hardcoded base64 in secrets.yml | Generated by Helm, auto-managed |
| **Storage** | Manual PV + PVC files | Configured via `persistence.size` value |
| **Configuration** | Edit yaml files | `--set` or values.yaml |
| **Replicas** | Hardcoded in yaml | `--set primary.replicaCount=3` |
| **Metrics** | Not included | `--set metrics.enabled=true` |
| **Versioning** | Manual git tags | `helm history` and `helm rollback` |
| **Environments (dev/prod)** | Duplicate all files with different values | Single chart, different values per env |
| **Rollback** | Manual git revert + kubectl apply | `helm rollback <release> <revision>` |
| **Charts as dependencies** | Import via git submodules (hacky) | Native support with Chart.yaml |
| **Community charts** | Build from scratch | Use existing Bitnami, Jetstack, etc. |

### Time Comparison

**Raw YAML Approach (First Deployment):**
```
Write deployment.yml        → 20 min
Write service.yml          → 10 min
Write secrets.yml          → 10 min
Write pvc.yml              → 10 min
Write pv.yml               → 10 min
Debug and test             → 30 min
Total: ~90 minutes
```

**Helm Chart Approach (First Deployment):**
```
Add repo + search           → 2 min
helm install command       → 1 min
Debug and test             → 5 min
Total: ~8 minutes
```

**Helm Advantage: ~80 minutes saved on first deployment!**

---

## Why AI-BankApp Needs Helm

### Current Problem

The AI-BankApp's k8s/ directory has:
1. **12 separate YAML files** - hard to manage
2. **Hardcoded values** - can't reuse across environments
3. **No versioning** - no rollback capability
4. **Manual dependencies** - MySQL config is separate from app config
5. **Scaling difficulties** - need to edit multiple files to scale

### With Helm Chart

- **Single chart** - all 12 components in one unit
- **Environment-specific values** - values-dev.yaml, values-prod.yaml
- **Automatic versioning** - each release tracked
- **Dependency management** - app chart depends on MySQL chart
- **Easy scaling** - `helm upgrade --set replicaCount=5`
- **CI/CD friendly** - deployment from GitOps pipeline
- **Community adoption** - share chart on ArtifactHub

---

## Commands Summary

```bash
# Repository Management
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/mysql

# Install/Deploy
helm install <release-name> <chart-name> --set key=value
helm install <release-name> <chart-name> -f values.yaml

# Check Status
helm list
helm list -A                    # All namespaces
helm status <release-name>
helm get values <release-name>

# Update/Upgrade
helm upgrade <release-name> <chart-name> --set key=value
helm upgrade --install <release-name> <chart-name>  # Install or upgrade

# History & Rollback
helm history <release-name>
helm rollback <release-name> <revision>

# Debugging
helm template <release-name> <chart-name>     # Render without deploying
helm get manifest <release-name>              # View deployed manifest

# Cleanup
helm uninstall <release-name>
```

---

## Key Takeaways

1. **Helm is a package manager** - Simplifies Kubernetes deployments
2. **Charts are reusable units** - One chart, many environments
3. **Values enable customization** - No need to edit YAML files
4. **Releases are versioned** - Easy rollback with `helm rollback`
5. **Community charts save time** - Use Bitnami, Jetstack, etc.
6. **Tomorrow's task** - Convert AI-BankApp's 12 YAML files into a custom Helm chart

---

## Screenshots/Evidence

> Screenshots would be added here after completing practical tasks:
> - `helm list` output showing deployed MySQL
> - `helm history` output showing revisions
> - MySQL pod running: `kubectl get pods -l app=mysql`
> - Successful MySQL connection: `kubectl exec -it mysql-0 -- mysql -u root -p`
> - Chart structure after `helm pull --untar`

---

## Resources

- [Official Helm Documentation](https://helm.sh/docs/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Bitnami Charts](https://charts.bitnami.com/)
- [ArtifactHub - Chart Marketplace](https://artifacthub.io/)
- [Helm Chart Security](https://helm.sh/docs/topics/security/)

---

## Next Steps (Day 79)

- Build a custom Helm chart for the AI-BankApp
- Convert all 12 raw YAML files into templates
- Create values files for dev, staging, and production
- Test chart with `helm template` and `helm install`
- Push chart to a registry
