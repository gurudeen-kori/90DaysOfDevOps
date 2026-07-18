# Day 80: Helm Project - Multi-Environment Deployment and CI/CD

## Overview

This project brings together three days of Helm learning into a production-ready deployment strategy for the AI-BankApp. We create environment-specific configurations for dev, staging, and production, implement Helm hooks for pre-deployment validation, package the chart for distribution, and integrate Helm into a GitOps CI/CD pipeline.

**Repository Reference:** [AI-BankApp-DevOps (feat/gitops branch)](https://github.com/TrainWithShubham/AI-BankApp-DevOps)

---

## Task 1: Environment-Specific Values Files

### The Strategy: One Chart, Three Environments

The power of Helm lies in using a single chart with different values files for each environment. This approach ensures:
- **Consistency**: Same logic across all environments
- **Maintainability**: Single source of truth for chart structure
- **Scalability**: Easy to add new environments

### values-dev.yaml (Development)

**Purpose**: Minimal resources for local/Kind cluster development

```yaml
bankapp:
  replicaCount: 1                    # Single instance for dev
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "latest"                    # Always use latest for fast iteration
    pullPolicy: Always               # Force re-pull on every restart
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "250m"
  autoscaling:
    enabled: false                   # No HPA in dev

mysql:
  enabled: true
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "250m"
  persistence:
    size: 2Gi                        # Minimal storage
    storageClass: standard

ollama:
  enabled: true
  model: tinyllama                   # Lightweight model for dev
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "1.5Gi"
      cpu: "1000m"
  persistence:
    size: 5Gi
    storageClass: standard

storageClass:
  create: false                      # Use cluster's default
```

**Key Decisions**:
- `tag: "latest"` enables rapid iteration during development
- `pullPolicy: Always` ensures developers get the latest code
- No HPA (Horizontal Pod Autoscaler) for simplicity
- Single replica to conserve resources on Kind cluster

### values-staging.yaml (Staging)

**Purpose**: Pre-production environment mirroring production patterns

```yaml
bankapp:
  replicaCount: 2                    # Multiple replicas for testing HPA
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"                    # Pinned version (tested in prod)
    pullPolicy: IfNotPresent         # Optimize image pulls
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 3
    targetCPUUtilization: 75         # Scale up at 75% CPU

mysql:
  enabled: true
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  persistence:
    size: 5Gi                        # Moderate storage for testing
    storageClass: gp3                # AWS gp3 volume

ollama:
  enabled: true
  model: tinyllama
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: StagingPass@456 # Staging credentials
  mysqlUser: root
  mysqlPassword: StagingPass@456

storageClass:
  create: true                       # Create staging storage class
```

**Key Decisions**:
- HPA enabled to test autoscaling logic before production
- Pinned image tag for consistent test environment
- Medium resource allocation to test realistic conditions
- Staging-specific credentials (never reuse production secrets)

### values-prod.yaml (Production)

**Purpose**: Highly available, scalable production configuration

```yaml
bankapp:
  replicaCount: 4                    # Multiple replicas for high availability
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"                    # Pinned, tested version only
    pullPolicy: IfNotPresent         # Optimize image pulls
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 4
    targetCPUUtilization: 70         # Scale at 70% (conservative)

mysql:
  enabled: true
  resources:
    requests:
      memory: "512Mi"                # Higher resource allocation
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  persistence:
    size: 20Gi                       # Large persistent storage
    storageClass: gp3

ollama:
  enabled: true
  model: tinyllama
  resources:
    requests:
      memory: "2Gi"                  # ML model requires more memory
      cpu: "900m"
    limits:
      memory: "2.5Gi"
      cpu: "1500m"
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: ProdSecure@789  # Production credentials
  mysqlUser: root
  mysqlPassword: ProdSecure@789

storageClass:
  create: true

gateway:
  enabled: true                      # Enable gateway only in prod
```

**Key Decisions**:
- Higher replica count for fault tolerance
- Conservative HPA thresholds (70% vs 75%) for stability
- Large resource allocations for predictable performance
- Gateway enabled only in production (not needed in lower environments)
- Production-grade storage and credentials

### Environment Comparison Table

| Setting | Dev | Staging | Prod |
|---------|-----|---------|------|
| **BankApp Replicas** | 1 (fixed) | 2–3 (HPA) | 2–4 (HPA) |
| **Image Tag** | latest | v1.2.0 | v1.2.0 |
| **Image Pull Policy** | Always | IfNotPresent | IfNotPresent |
| **App Memory Req/Limit** | 256Mi / 512Mi | 256Mi / 512Mi | 256Mi / 512Mi |
| **App CPU Req/Limit** | 100m / 250m | 250m / 500m | 250m / 500m |
| **MySQL Memory Req/Limit** | 128Mi / 256Mi | 256Mi / 512Mi | 512Mi / 1Gi |
| **MySQL Storage** | 2Gi | 5Gi | 20Gi |
| **MySQL Storage Class** | standard | gp3 | gp3 |
| **Ollama Memory Req/Limit** | 1Gi / 1.5Gi | 2Gi / 2Gi | 2Gi / 2.5Gi |
| **Ollama Storage** | 5Gi | 10Gi | 10Gi |
| **HPA Enabled** | false | true | true |
| **HPA Target CPU** | — | 75% | 70% |
| **Gateway Enabled** | false | false | true |
| **Storage Class Create** | false | true | true |

### Deployment Commands

#### Dev (on Kind cluster)
```bash
helm install bankapp-dev bankapp/ \
  -f bankapp/values-dev.yaml \
  -n dev --create-namespace
```

#### Staging (dry-run to verify)
```bash
# Render templates to inspect
helm template bankapp-staging bankapp/ \
  -f bankapp/values-staging.yaml | grep "replicas:"

# Expected output: replicas: 2
```

#### Production (dry-run to verify)
```bash
helm template bankapp-prod bankapp/ \
  -f bankapp/values-prod.yaml | grep "replicas:"

# Expected output: replicas: 4
```

---

## Task 2: Helm Hooks for Pre-Deployment Validation

### Why Helm Hooks?

Helm hooks allow us to run Kubernetes jobs or pods at specific points in the release lifecycle:
- **pre-install**: Before the chart is deployed
- **post-install**: After successful deployment
- **pre-upgrade**: Before upgrading an existing release
- **post-upgrade**: After upgrade completes
- **pre-delete**: Before uninstalling the chart
- **test**: When running `helm test`

For the AI-BankApp, we use hooks to ensure MySQL is ready before deploying the application.

### Pre-Install Hook: Database Readiness Check

**File**: `bankapp/templates/pre-install-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "bankapp.fullname" . }}-db-ready
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      containers:
        - name: db-check
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - |
              echo "Waiting for MySQL to be ready..."
              until nc -z {{ include "bankapp.fullname" . }}-mysql 3306; do
                echo "MySQL not ready, retrying in 3s..."
                sleep 3
              done
              echo "MySQL is ready!"
          resources:
            requests: { memory: "32Mi", cpu: "50m" }
            limits: { memory: "64Mi", cpu: "100m" }
      restartPolicy: Never
  backoffLimit: 10
```

**Annotation Breakdown**:

| Annotation | Value | Purpose |
|-----------|-------|---------|
| `helm.sh/hook` | pre-install,pre-upgrade | Runs before install AND before upgrade |
| `helm.sh/hook-weight` | 0 | Execution order (lower runs first) |
| `helm.sh/hook-delete-policy` | before-hook-creation | Clean up old hook job before creating new one |

**What the Hook Does**:
1. Runs a busybox container with `nc` (netcat) to check MySQL connectivity
2. Retries every 3 seconds until MySQL port 3306 responds
3. Prevents deployment from proceeding until MySQL is ready
4. Fails after 10 retries (backoffLimit) if MySQL remains down

**Defense in Depth**:
- This hook handles the cluster startup phase
- Init containers in the Deployment handle runtime connectivity issues
- Combined approach ensures maximum resilience

### Helm Test: Verify Application Health

**File**: `bankapp/templates/tests/test-connection.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "bankapp.fullname" . }}-test
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ['sh', '-c', 'wget -qO- http://{{ include "bankapp.fullname" . }}-service:8080/actuator/health']
  restartPolicy: Never
```

**What the Test Does**:
1. Creates a temporary pod that runs after deployment
2. Makes HTTP GET to Spring Boot `/actuator/health` endpoint
3. Pod succeeds if the app is running and healthy

**Running the Test**:
```bash
helm test bankapp-dev -n dev

# Output:
# Pod bankapp-dev-test pending
# Pod bankapp-dev-test succeeded
# NAME: bankapp-dev
# LAST DEPLOYED: Sat Jul 18 10:30:45 2026
# NAMESPACE: dev
# STATUS: deployed
# REVISION: 1
# TEST SUITE: bankapp-dev-test
# PASSED
```

### Other Useful Hook Types

**post-install** - Run database migrations:
```yaml
annotations:
  "helm.sh/hook": post-install
  "helm.sh/hook-weight": "1"
```

**pre-delete** - Backup database before teardown:
```yaml
annotations:
  "helm.sh/hook": pre-delete
  "helm.sh/hook-weight": "0"
```

---

## Task 3: Package and Version the Chart

### Linting and Validation

Before packaging, always lint the chart:

```bash
helm lint bankapp/

# Output:
# ==> Linting bankapp
# [OK] Chart.yaml is valid
# [OK] Chart.yaml 'version' is valid
# [OK] Chart.yaml 'appVersion' is valid
# [OK] values.yaml is valid
# [OK] Chart.lock is valid
# [OK] templates are valid
```

### Initial Packaging

```bash
helm package bankapp/

# Creates: bankapp-0.1.0.tgz
# Includes entire chart directory, compressed
```

### Version Bumping

Edit `bankapp/Chart.yaml`:

```yaml
apiVersion: v2
name: bankapp
description: AI-BankApp Helm Chart
type: application
version: 0.2.0                  # Bumped from 0.1.0 (chart structure changed)
appVersion: "1.1.0"             # Bumped from 1.0.0 (app version updated)
```

**Semantic Versioning Guide**:
- **MAJOR.MINOR.PATCH**
- **0.1.0** → **0.2.0**: New features (hooks added)
- **1.0.0** → **1.1.0**: New app features
- **1.0.0** → **1.0.1**: Bug fix

### Re-package After Changes

```bash
helm package bankapp/

# Creates: bankapp-0.2.0.tgz
# Now you have both: bankapp-0.1.0.tgz and bankapp-0.2.0.tgz
```

### Installing from a Packaged Chart

```bash
helm install my-bankapp bankapp-0.2.0.tgz \
  -f bankapp/values-prod.yaml \
  -n bankapp \
  --create-namespace
```

### Creating a Chart Repository

For distributing charts via GitHub Pages or Artifactory:

```bash
# Prepare repository directory
mkdir chart-repo
cp bankapp-*.tgz chart-repo/

# Generate index file
helm repo index chart-repo/ \
  --url https://your-username.github.io/helm-charts

# View the index
cat chart-repo/index.yaml
```

**Sample index.yaml**:
```yaml
apiVersion: v1
entries:
  bankapp:
  - apiVersion: v2
    appVersion: "1.1.0"
    created: "2026-07-18T10:30:45Z"
    description: AI-BankApp Helm Chart
    digest: abc123def456...
    name: bankapp
    type: application
    urls:
    - https://your-username.github.io/helm-charts/bankapp-0.2.0.tgz
    version: 0.2.0
  - apiVersion: v2
    appVersion: "1.0.0"
    created: "2026-07-18T09:15:30Z"
    description: AI-BankApp Helm Chart
    digest: xyz789uvw012...
    name: bankapp
    type: application
    urls:
    - https://your-username.github.io/helm-charts/bankapp-0.1.0.tgz
    version: 0.1.0
generated: "2026-07-18T10:30:45Z"
```

### Sharing the Repository

```bash
# Add your repo for others to use
helm repo add ai-bankapp https://your-username.github.io/helm-charts
helm repo update

# Install from repository
helm install bankapp ai-bankapp/bankapp \
  -f values-prod.yaml \
  -n bankapp
```

---

## Task 4: Helm Integration with GitOps CI/CD Pipeline

### Current Pipeline (Raw Manifests)

The AI-BankApp currently uses raw Kubernetes manifests in the `k8s/` directory:

1. Developer pushes code to GitHub
2. GitHub Actions builds Docker image → tags with git SHA
3. Updates image tag in `k8s/bankapp-deployment.yml` (via sed)
4. Commits change back to repository
5. ArgoCD detects change and syncs to EKS

**Limitation**: Raw manifests lack templating, leading to YAML duplication across environments.

### New Pipeline (With Helm)

With Helm integration, the pipeline becomes cleaner:

1. Developer pushes code to GitHub
2. GitHub Actions builds Docker image → tags with git SHA
3. Updates `bankapp.image.tag` in `helm-chart/bankapp/values-prod.yaml`
4. Commits change back to repository
5. ArgoCD detects change and runs `helm upgrade` on EKS

### GitHub Actions Workflow Integration

**Workflow File**: `.github/workflows/helm-cd.yml`

```yaml
name: Helm CD Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'helm-chart/**'
      - '.github/workflows/helm-cd.yml'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build Docker Image
        run: |
          docker build -t trainwithshubham/ai-bankapp-eks:${{ github.sha }} .
          docker push trainwithshubham/ai-bankapp-eks:${{ github.sha }}

      - name: Get Short SHA
        id: tag
        run: |
          echo "sha_short=$(git rev-parse --short ${{ github.sha }})" >> $GITHUB_OUTPUT

      - name: Update Helm Values
        run: |
          # Install yq if not present
          sudo apt-get update && sudo apt-get install -y yq
          
          # Update image tag in values-prod.yaml
          yq -i '.bankapp.image.tag = "${{ steps.tag.outputs.sha_short }}"' \
            helm-chart/bankapp/values-prod.yaml

      - name: Commit Updated Values
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add helm-chart/bankapp/values-prod.yaml
          git diff --staged --quiet || \
            git commit -m "ci: update bankapp image to ${{ steps.tag.outputs.sha_short }} [skip ci]"
          git push

      - name: Helm Lint
        run: |
          helm lint helm-chart/bankapp/

      - name: Helm Dry-Run
        run: |
          helm upgrade --install bankapp-dry-run helm-chart/bankapp/ \
            -f helm-chart/bankapp/values-prod.yaml \
            --dry-run --debug
```

**Key Features**:
- Uses `yq` to update YAML safely (preserves structure)
- Commits only if values changed (`--quiet` check)
- `[skip ci]` flag prevents infinite CI loops
- Dry-run validates syntax before ArgoCD deploys

### ArgoCD ApplicationSet Configuration

**File**: `argocd/bankapp-application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/AI-BankApp-DevOps.git
    targetRevision: main
    path: helm-chart/bankapp
    helm:
      releaseName: bankapp
      valueFiles:
        - values-prod.yaml              # Use prod values
      values: |                         # Override specific values
        bankapp:
          image:
            tag: "1.2.0"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Advantages Over Raw Manifests**:
- ArgoCD natively renders Helm templates before syncing
- Supports multiple value files for different environments
- Tracks drift against rendered Helm output
- Cleaner YAML structure in Git

### Pipeline Flow Diagram

```
Developer Push
    ↓
GitHub Actions (Build & Update)
    - Build Docker Image
    - Tag with Git SHA
    - Update values-prod.yaml
    - Commit & Push
    ↓
ArgoCD Detects Change
    - Polls Git repository
    - Compares desired state
    ↓
Helm Upgrade
    - Renders templates
    - Merges values files
    - Applies manifests
    ↓
Production EKS Cluster
    - Updates running pods
    - Maintains rollback capability
```

### Advantages of Helm + ArgoCD in GitOps

| Aspect | Raw Manifests | Helm + ArgoCD |
|--------|---------------|---------------|
| **Configuration Reuse** | Copy-paste across envs | One chart, many values files |
| **Environment Drift** | Manual values in manifests | Centralized values files |
| **Templating** | None (use kustomize) | Built-in Go templating |
| **Version Control** | Track all manifests | Track values & versions |
| **Rollback** | Re-deploy old manifest | `helm rollback` + ArgoCD sync |
| **Dependencies** | Manual management | Chart dependencies in Chart.yaml |
| **Testing** | `kubectl dry-run` | `helm template` + `helm test` |

---

## Task 5: Helm Best Practices for Production

### 1. Always Use `helm upgrade --install`

This pattern creates the release if it doesn't exist, or upgrades if it does:

```bash
helm upgrade --install bankapp \
  bankapp/ \
  -f bankapp/values-prod.yaml \
  --set bankapp.image.tag=$GIT_SHA \
  -n bankapp \
  --create-namespace \
  --wait \
  --timeout 300s \
  --atomic
```

**Flag Breakdown**:

| Flag | Purpose |
|------|---------|
| `--install` | Create release if missing |
| `-f values-prod.yaml` | Use production values |
| `--set` | Override specific values (highest priority) |
| `--create-namespace` | Create namespace if it doesn't exist |
| `--wait` | Block until all pods are ready |
| `--timeout 300s` | Wait max 5 minutes for deployment |
| `--atomic` | Auto-rollback on failure (all-or-nothing) |

**Why `--atomic` Matters in CI/CD**:
Without `--atomic`, a failed deployment leaves the cluster in a broken state. With it, the previous working version is restored automatically.

### 2. Use `helm diff` Before Upgrading

The helm-diff plugin shows exactly what would change:

```bash
# Install the plugin
helm plugin install https://github.com/databus23/helm-diff

# Preview changes
helm diff upgrade bankapp bankapp/ \
  -f bankapp/values-prod.yaml \
  -n bankapp

# Output shows:
# == deployment.apps/bankapp ==
# - replicas: 3
# + replicas: 4
# ...other changes...
```

**Best Practice**: Always run `helm diff` before `helm upgrade` in production.

### 3. Resource Quotas per Namespace

Prevent a single application from consuming cluster resources:

**File**: `bankapp/templates/resourcequota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {{ include "bankapp.fullname" . }}-quota
  namespace: {{ .Release.Namespace }}
spec:
  hard:
    requests.cpu: "2"                # Max 2 CPU cores requested
    requests.memory: 4Gi              # Max 4GB memory requested
    limits.cpu: "4"                  # Max 4 CPU cores limited
    limits.memory: 8Gi               # Max 8GB memory limited
    pods: "20"                       # Max 20 pods in namespace
    services: "5"                    # Max 5 services
    persistentvolumeclaims: "3"      # Max 3 PVCs
```

**Enable via values.yaml**:
```yaml
resourceQuota:
  enabled: true
```

### 4. Secret Management in Production

**NEVER store real secrets in values.yaml**. Instead, use external secret systems:

#### Option A: AWS Secrets Manager + External Secrets Operator

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace
```

**Template**: `bankapp/templates/externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: {{ .Release.Namespace }}
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

---

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "bankapp.fullname" . }}-secrets
  namespace: {{ .Release.Namespace }}
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: mysql-secret
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: bankapp/mysql/username
    - secretKey: password
      remoteRef:
        key: bankapp/mysql/password
```

#### Option B: Sealed Secrets

```bash
# Install Sealed Secrets controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets \
  sealed-secrets/sealed-secrets \
  -n kube-system
```

#### Option C: HashiCorp Vault

```yaml
# Use Vault Agent Injector
vault.hashicorp.com/agent-inject: "true"
vault.hashicorp.com/agent-inject-secret-database: "secret/data/bankapp/mysql"
vault.hashicorp.com/role: "bankapp-prod"
```

### 5. Production Checklist

Before deploying to production:

- [ ] All images use pinned version tags (not `latest`)
- [ ] Resource requests and limits are defined for all containers
- [ ] Liveness and readiness probes are configured
- [ ] PodDisruptionBudget is set for high availability
- [ ] NetworkPolicy restricts pod-to-pod communication
- [ ] Secrets are externalized (not in values.yaml)
- [ ] Resource quotas limit namespace consumption
- [ ] Helm hooks validate dependencies (pre-install job)
- [ ] Backup strategy for persistent data is documented
- [ ] Monitoring and alerting are configured
- [ ] Rollback procedure has been tested
- [ ] Security scanning passed on Docker images

---

## Task 5: Clean Up and Review

### Check Deployments

```bash
helm list -A

# Output:
# NAME          NAMESPACE   REVISION  UPDATED                    STATUS   CHART           APP VERSION
# bankapp-dev   dev         1         2026-07-18 10:30:45 +0000  deployed  bankapp-0.2.0   1.1.0
```

### Get Release Details

```bash
helm get values bankapp-dev -n dev
helm get manifest bankapp-dev -n dev
helm history bankapp-dev -n dev
```

### Uninstall Releases

```bash
# Uninstall from dev
helm uninstall bankapp-dev -n dev

# Delete the namespace
kubectl delete namespace dev

# Destroy Kind cluster
kind delete cluster --name tws-cluster
```

### 3-Day Helm Learning Journey

| Day | Concept | AI-BankApp Application |
|-----|---------|----------------------|
| **78** | Helm basics: repos, install, upgrade, rollback | Deployed MySQL via Bitnami Helm chart |
| **79** | Custom chart creation, Go templating | Converted 12 raw Kubernetes manifests into reusable Helm chart |
| **80** | Multi-env values, hooks, packaging, CI/CD | Built production-ready chart with dev/staging/prod configs, integrated with GitOps |

### When to Use What

| Tool | Best For | When NOT to Use |
|------|----------|----------------|
| **Raw Manifests** | Simple single-env deployments | Complex multi-env apps (leads to copy-paste) |
| **Helm** | Multi-env, complex apps with dependencies | Simple single-service deployments (overhead) |
| **Kustomize** | Layered patches on existing manifests | Need templating and package management |
| **Helm + Kustomize** | Maximum flexibility | Excessive complexity, hard to debug |

**For AI-BankApp**: Helm is the ideal choice because:
- Multi-environment (dev, staging, prod)
- Multiple components (MySQL, Ollama, BankApp)
- Complex dependencies (hooks, init containers)
- Shared GitOps pipeline (ArgoCD integration)

---

## Key Insights

### Environment Scaling Patterns

**Development** focuses on minimal resource usage:
- Single replica (no redundancy needed)
- Latest image tag (fast iteration)
- Simple storage (standard class)

**Staging** mirrors production patterns:
- Multiple replicas with HPA
- Pinned image versions (test specific builds)
- Production storage classes (verify performance)

**Production** prioritizes availability and performance:
- High replica counts with aggressive HPA
- Pinned, tested versions
- Large storage and resources
- Additional services (gateway)

### Helm Hooks Enable Sophisticated Patterns

Pre-install hooks solve the "chicken-and-egg" problem:
- **Without hooks**: Init containers wait for MySQL, but deployment proceeds even if MySQL isn't ready
- **With hooks**: Helm won't create the Deployment until MySQL is confirmed ready

This defense-in-depth approach (hooks + init containers) ensures maximum reliability.

### GitOps Benefits Realized Through Helm

With Helm in a GitOps pipeline:
1. **Git is the source of truth** for all configurations
2. **Reproducible deployments** (same commit = same cluster state)
3. **Easy rollbacks** (revert Git commit, ArgoCD syncs)
4. **Audit trail** (who changed what and when)
5. **Testing** (helm template and helm test in CI)

---

## Next Steps

1. **Extend the Chart**:
   - Add Prometheus monitoring
   - Implement cert-manager integration
   - Add Ingress configuration

2. **Advanced Features**:
   - Chart museum for private registry
   - Helm security scanning (chartmuseum, artifacthub)
   - Multi-chart deployments (umbrella charts)

3. **Integration**:
   - Connect ArgoCD to your cluster
   - Set up GitOps workflow
   - Implement automated testing in pipeline

4. **Documentation**:
   - Document Helm values with comments
   - Create troubleshooting guide
   - Record runbooks for common operations

---

## References

- [Helm Official Documentation](https://helm.sh/docs/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [ArgoCD Helm Integration](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [AI-BankApp DevOps Repository](https://github.com/TrainWithShubham/AI-BankApp-DevOps)

---

**Documentation Date**: July 18, 2026  
**Author**: DevOps Training - Day 80 Project  
**Status**: Production Ready
