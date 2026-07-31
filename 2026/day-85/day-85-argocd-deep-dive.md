# Day 85: ArgoCD Deep Dive - Sync Strategies, Rollbacks, and Multi-App Management

**Date**: 2026-04-10  
**Topic**: Production-grade ArgoCD patterns for managing multiple applications  
**Reference**: [AI-BankApp-DevOps](https://github.com/TrainWithShubham/AI-BankApp-DevOps) (feat/gitops branch)

---

## 📋 Overview

Yesterday we deployed the AI-BankApp via ArgoCD and tested self-healing. Today we explore the patterns that production teams use to manage dozens of applications across Kubernetes clusters:

- **Sync Strategies**: Automated vs manual sync with preview capabilities
- **Sync Waves**: Ordered resource deployment with dependencies
- **Rollbacks**: Revision history and recovery strategies
- **App of Apps Pattern**: Multi-application management at scale
- **Notifications**: Alert integration for deployment events
- **Projects & RBAC**: Multi-tenancy and access control

---

## 1. Sync Strategies: Automated vs Manual

### 1.1 Automated Sync (Development/Staging)

**Use Case**: Dev and staging environments where continuous deployment is desired

```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources removed from Git
    selfHeal: true   # Revert manual cluster changes
```

**Characteristics**:
- Every Git change syncs automatically (default: 3-minute interval)
- No human approval needed
- ArgoCD reverts manual cluster changes (self-healing)
- Faster feedback loop for developers
- Less safe for production

**Workflow**:
1. Developer pushes code to Git
2. ArgoCD detects change within 3 minutes
3. Application syncs automatically
4. Self-healing continuously corrects drift

---

### 1.2 Manual Sync (Production)

**Use Case**: Production environments requiring approval gates

```yaml
syncPolicy: {}   # No automated section
```

**Characteristics**:
- ArgoCD detects drift but does NOT auto-correct
- Human must explicitly click "Sync" or run `argocd app sync`
- Allows preview of changes before applying
- Better control and audit trail
- Prevents accidental deployments

**Workflow**:
1. Developer pushes code to Git
2. ArgoCD detects drift (OutOfSync status)
3. Team reviews changes in UI or CLI
4. On approval, human syncs the application
5. Changes apply with full audit log

---

### 1.3 Switch from Automated to Manual

```bash
# Disable automated sync
argocd app set bankapp --sync-policy none

# Verify status
argocd app get bankapp
# Output shows: OutOfSync status detected but NOT applied

# Preview what would change (dry-run)
argocd app diff bankapp

# Show sync preview with resources
argocd app sync bankapp --dry-run

# Sync manually (after review)
argocd app sync bankapp

# Re-enable automated sync
argocd app set bankapp --sync-policy automated --self-heal --auto-prune
```

---

### 1.4 Comparison Table

| Aspect | Automated | Manual |
|--------|-----------|--------|
| **Sync Trigger** | Automatic every 3 min | Human clicks "Sync" |
| **Self-Healing** | Yes (reverts drift) | No (detects only) |
| **Resource Pruning** | Yes (deletes removed resources) | No |
| **Approval Gate** | None | Required |
| **Use Case** | Dev, Staging | Production |
| **Safety** | Lower (fast feedback) | Higher (controlled) |
| **Audit Trail** | Automatic | Explicit sync events |
| **Recovery Speed** | Fast | Slower (manual approval) |

---

## 2. Sync Waves and Resource Ordering

### 2.1 The Problem: Dependencies

The AI-BankApp has dependencies:
- Namespace must exist before creating resources
- StorageClass must exist before PVCs
- PVCs must be bound before MySQL pod uses them
- MySQL must be ready before BankApp connects
- Services must exist before HPA targets them

**Without sync waves**: ArgoCD might try to create everything simultaneously, causing race conditions and deployment failures.

**With sync waves**: Resources are created in a specific order, ensuring dependencies are satisfied.

---

### 2.2 Sync Wave Annotations

Sync waves use integer annotations to control deployment order:
- Negative numbers execute first
- Zero executes in the middle
- Positive numbers execute last
- Resources in the same wave deploy in parallel
- ArgoCD waits for each wave to be healthy before proceeding

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

---

### 2.3 AI-BankApp Sync Wave Configuration

| Resource | Wave | Reason |
|----------|------|--------|
| **Namespace** | `-2` | Infrastructure foundation |
| **StorageClass** | `-2` | Required for PVCs |
| **PVCs** | `-1` | Prepared before MySQL mounts |
| **ConfigMap** | `-1` | Configuration ready early |
| **Secret** | `-1` | Credentials available early |
| **MySQL Deployment** | `0` | Database ready before app |
| **Ollama Deployment** | `0` | LLM service ready (parallel) |
| **Services** | `0` | Networking available (parallel) |
| **BankApp Deployment** | `1` | Waits for MySQL and services |
| **HPA** | `2` | Autoscaling after app stable |

---

### 2.4 Implementation: Adding Sync Wave Annotations

Edit each Kubernetes manifest in `k8s/`:

**k8s/namespace.yml** (Wave -2):
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bankapp
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
```

**k8s/pv.yml - StorageClass** (Wave -2):
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
provisioner: ebs.csi.aws.com
```

**k8s/pvc.yml** (Wave -1):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  ...
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-pvc
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  ...
```

**k8s/configmap.yml** (Wave -1):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bankapp-config
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
data:
  APP_NAME: "AI-BankApp"
  MYSQL_HOST: "mysql"
```

**k8s/secrets.yml** (Wave -1):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bankapp-secrets
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
type: Opaque
data:
  DB_PASSWORD: <base64-encoded>
```

**k8s/mysql-deployment.yml** (Wave 0):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  ...
```

**k8s/ollama-deployment.yml** (Wave 0):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  ...
```

**k8s/service.yml - All three services** (Wave 0):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  ...
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  ...
---
apiVersion: v1
kind: Service
metadata:
  name: bankapp
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  ...
```

**k8s/bankapp-deployment.yml** (Wave 1):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bankapp
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  template:
    spec:
      initContainers:
      - name: wait-for-mysql
        image: busybox:1.28
        command: ['sh', '-c', 'until nc -z mysql 3306; do sleep 1; done']
      containers:
      - name: bankapp
        image: bankapp:latest
        env:
        - name: MYSQL_HOST
          valueFrom:
            configMapKeyRef:
              name: bankapp-config
              key: MYSQL_HOST
```

**k8s/hpa.yml** (Wave 2):
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: bankapp-hpa
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: bankapp
  minReplicas: 2
  maxReplicas: 10
```

---

### 2.5 Deployment Timeline Visualization

```
Time →

[Wave -2] Namespace, StorageClass
          ↓ (waits for healthy)
[Wave -1] PVCs, ConfigMap, Secret
          ↓ (waits for healthy)
[Wave  0] MySQL, Ollama, Services (parallel)
          ↓ (waits for healthy)
[Wave  1] BankApp Deployment
          ↓ (waits for healthy)
[Wave  2] HPA

Result: Ordered deployment ensuring all dependencies are met
```

---

### 2.6 Verification

After pushing sync wave annotations:

```bash
# Watch the sync progress
kubectl get pods -n bankapp -w

# Check sync wave timing in ArgoCD
argocd app get bankapp --refresh

# View detailed sync information
argocd app get bankapp --show-operation
```

In the ArgoCD UI:
- Sync tab shows wave progress
- Resources are grouped by wave number
- Visual indicator shows which wave is currently syncing

---

## 3. ArgoCD Rollbacks

### 3.1 The Problem

Deployment fails, and you need to quickly revert to a known good state. ArgoCD tracks every sync as a **revision**.

### 3.2 Revision History

```bash
# View all previous syncs/revisions
argocd app history bankapp

# Output:
# ID  DATE                 REVISION
# 1   2026-04-10 10:00:00  abc1234567890 (Initial deployment)
# 2   2026-04-10 10:15:00  def5678901234 (Added sync waves)
# 3   2026-04-10 10:30:00  ghi2345678901 (Updated replica count)
# 4   2026-04-10 10:45:00  jkl6789012345 (BROKEN - bug introduced)
```

### 3.3 Rollback Methods

#### 3.3a CLI Rollback

```bash
# Rollback to revision 3 (the last known good state)
argocd app rollback bankapp 3

# Immediate effect
argocd app get bankapp
# Status: OutOfSync (cluster now at revision 3, HEAD is still at revision 4)
```

#### 3.3b UI Rollback

1. Open ArgoCD UI
2. Click Application → **History** tab
3. Select previous revision
4. Click **Rollback**

### 3.4 What Rollback Does (and Doesn't Do)

✅ **Rollback DOES**:
- Update the cluster to match an older Git commit
- Instantly reverts running pods and services
- Fast recovery from production incidents

❌ **Rollback DOES NOT**:
- Change Git history
- Update the ArgoCD Application spec
- Create an audit trail of the revert

### 3.5 The GitOps-Correct Approach: Git Revert

**Rollback gets you out of the fire. Git Revert keeps you safe.**

```bash
# Check current HEAD
git log --oneline | head -5
# Output:
# jkl6789 (HEAD) Update replica count - BROKEN
# ghi2345 Updated replica count - working
# def5678 Added sync waves
# abc1234 Initial deployment

# Revert the last commit (creates a NEW commit that undoes it)
git revert HEAD

# Git opens an editor - write a clear message
# "Revert: Update replica count - caused pod crashes"

# Push the revert
git push

# ArgoCD syncs the revert within 3 minutes
argocd app wait bankapp
```

**Result**: 
- Cluster is now at a known good state
- Git history shows the complete story: commit + revert
- Every change is auditable
- No mystery about why deployments were rolled back

### 3.6 Comparison: Rollback vs Revert

| Aspect | ArgoCD Rollback | Git Revert |
|--------|-----------------|-----------|
| **Speed** | Immediate | 3 min (sync delay) |
| **Git History** | Not changed | New commit added |
| **Audit Trail** | ArgoCD history only | Git log shows everything |
| **Permanent** | Temporary (until next sync) | Permanent (in Git) |
| **Use Case** | Emergency (quick recovery) | Production (proper fix) |
| **Follow-up** | git revert needed | Already done |
| **Best Practice** | Do this THEN git revert | Use this for lasting fix |

### 3.7 Production Incident Recovery Workflow

```
Step 1: DETECT
- Monitor alerts or ArgoCD health status
- Confirm deployment is broken

Step 2: IMMEDIATE RECOVERY (ArgoCD Rollback)
argocd app rollback bankapp 3
# Service restored within seconds

Step 3: ROOT CAUSE ANALYSIS
- Review what changed in revision 4
- Identify the bug

Step 4: PROPER FIX (Git Revert)
git revert HEAD    # Revert the bad commit
git push          # Push the revert
# ArgoCD syncs automatically (3 min)

Step 5: FIX THE BUG
- Create new branch
- Fix the underlying issue
- Submit PR for review
- Merge after approval

Step 6: VERIFY
argocd app get bankapp
# Status: Synced
# Cluster reflects the fix
```

---

## 4. App of Apps Pattern

### 4.1 The Problem

Managing a single application with ArgoCD is straightforward. Managing 50+ applications requires:
- A way to organize Applications
- Ability to add/remove apps easily
- Centralized configuration
- Consistency across deployments

**Solution**: The **App of Apps pattern** -- one parent Application that manages child Applications.

### 4.2 Architecture

```
┌─────────────────────────────────────┐
│        Root Application             │
│  (argocd-apps/root-app.yaml)       │
└──────────────┬──────────────────────┘
               │
               ├─→ Child App 1: bankapp
               ├─→ Child App 2: monitoring
               ├─→ Child App 3: envoy-gateway
               └─→ Child App N: ...
```

The root app watches a Git directory (`argocd-apps/`). Each YAML file in that directory is an Application manifest. ArgoCD creates all child applications automatically.

### 4.3 Implementation

#### 4.3a Create the Directory Structure

```bash
mkdir -p argocd-apps/
```

#### 4.3b Create Child Application: BankApp

**File**: `argocd-apps/bankapp.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: bankapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**Key Fields**:
- `metadata.name`: Unique application name
- `source.repoURL`: Git repository (can be any repo)
- `source.path`: Directory within repo containing manifests
- `destination.namespace`: Target namespace in cluster
- `finalizers`: Ensures resources are deleted when app is deleted

#### 4.3c Create Child Application: Monitoring (Helm Chart)

**File**: `argocd-apps/monitoring.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: "65.*"
    helm:
      values: |
        grafana:
          adminPassword: admin123
          persistence:
            enabled: true
            size: 10Gi
        prometheus:
          prometheusSpec:
            retention: 3d
            storageSpec:
              volumeClaimTemplate:
                spec:
                  accessModes: ["ReadWriteOnce"]
                  resources:
                    requests:
                      storage: 50Gi
            resources:
              requests:
                memory: 512Mi
                cpu: 250m
              limits:
                memory: 1Gi
                cpu: 500m
        alertmanager:
          enabled: true
          config:
            global:
              resolve_timeout: 5m
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**Key Features**:
- Uses Helm chart from public repository
- Provides custom values for Grafana and Prometheus
- Automatically creates monitoring namespace
- Persistent storage configured

#### 4.3d Create Child Application: Envoy Gateway

**File**: `argocd-apps/envoy-gateway.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: envoy-gateway
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: docker.io/envoyproxy
    chart: gateway-helm
    targetRevision: "v1.4.*"
    helm:
      values: |
        gateway:
          replicas: 3
          resources:
            requests:
              memory: 256Mi
              cpu: 100m
            limits:
              memory: 512Mi
              cpu: 200m
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

#### 4.3e Create the Root Application

**File**: `argocd-apps/root-app.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: argocd-apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
```

**How it Works**:
1. Root app watches `argocd-apps/` directory in Git
2. Finds all `*.yaml` files (bankapp.yaml, monitoring.yaml, envoy-gateway.yaml)
3. Creates three child Applications automatically
4. Each child app syncs independently

### 4.4 Deployment

```bash
# Apply the root application
kubectl apply -f argocd-apps/root-app.yaml

# Wait for ArgoCD to create child applications
sleep 30

# List all applications managed by root-app
argocd app list

# Output:
# NAME               CLUSTER                         NAMESPACE     PROJECT  STATUS     HEALTH
# root-app           https://kubernetes.default.svc  argocd        default  Synced     Healthy
# bankapp            https://kubernetes.default.svc  bankapp       default  Synced     Healthy
# monitoring         https://kubernetes.default.svc  monitoring    default  Synced     Healthy
# envoy-gateway      https://kubernetes.default.svc  envoy-gateway-system  default  Synced  Healthy
```

### 4.5 Adding a New Application

To add a new application to the cluster:

1. Create a new YAML file in `argocd-apps/`:

```yaml
# argocd-apps/logging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: logging
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://grafana.github.io/helm-charts
    chart: loki-stack
    targetRevision: "2.10.*"
  destination:
    server: https://kubernetes.default.svc
    namespace: logging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

2. Commit and push:
```bash
git add argocd-apps/logging.yaml
git commit -m "Add logging stack (Loki) via App of Apps pattern"
git push
```

3. ArgoCD automatically detects the new file and creates the application (within 3 minutes).

### 4.6 App of Apps Advantages

| Advantage | Benefit |
|-----------|---------|
| **Single Source of Truth** | All apps configured in Git |
| **Easy Scaling** | Add app by adding YAML file |
| **Consistency** | All child apps follow same pattern |
| **Centralized Control** | Root app manages sync policy |
| **Multi-team** | Each team can own their app YAML |
| **Disaster Recovery** | Recreate entire platform from Git |
| **Audit Trail** | Complete Git history of platform |

---

## 5. ArgoCD Notifications

### 5.1 The Problem

Deployments happen, but nobody knows:
- Did the sync succeed or fail?
- Why did the application become unhealthy?
- Which team needs to investigate?

**Solution**: Notifications alert teams in real-time.

### 5.2 Notification Architecture

```
ArgoCD Application
        ↓ (event: sync succeeded/failed/drift)
Notification Controller
        ↓ (matches trigger)
Template Renderer
        ↓ (formats message)
Service (Slack, webhook, email)
        ↓
Team receives alert
```

### 5.3 Setup Notification Controller

```bash
# Check if notifications controller is running
kubectl get pods -n argocd -l app.kubernetes.io/component=notifications-controller

# Output (should show running pod):
# NAME                                          READY   STATUS    RESTARTS   AGE
# argocd-notifications-controller-abc123        1/1     Running   0          5d
```

If not running, it's included in ArgoCD 2.4+. Install with:
```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/release-1.8/manifests/install.yaml
```

### 5.4 Configure Notification Triggers and Templates

```bash
kubectl apply -n argocd -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # TRIGGERS: Define when notifications fire
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-sync-succeeded]
  
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]
  
  trigger.on-sync-status-unknown: |
    - when: app.status.operationState.phase in ['Unknown']
      send: [app-sync-unknown]
  
  # TEMPLATES: Format the message
  template.app-sync-succeeded: |
    message: |
      ✅ Application {{.app.metadata.name}} sync succeeded!
      Revision: {{.app.status.sync.revision}}
      Namespace: {{.app.metadata.namespace}}
      By: {{(call .repo.GetCommitMetadata .app.status.sync.revision).Author}}
      
      View in ArgoCD: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}
  
  template.app-sync-failed: |
    message: |
      ❌ Application {{.app.metadata.name}} sync FAILED!
      Revision: {{.app.status.sync.revision}}
      Error: {{.app.status.operationState.phase}}
      Namespace: {{.app.metadata.namespace}}
      
      Check logs immediately: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}
  
  template.app-health-degraded: |
    message: |
      ⚠️ Application {{.app.metadata.name}} health is DEGRADED!
      Current Status: {{.app.status.health.status}}
      Namespace: {{.app.metadata.namespace}}
      
      Healthy Resources: {{.app.status.resources | filter "health.status" "Healthy" | len}}/{{.app.status.resources | len}}
      
      Investigate: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}
  
  template.app-sync-unknown: |
    message: |
      ❓ Application {{.app.metadata.name}} sync status UNKNOWN
      Last sync: {{.app.status.lastSyncTime}}
      
      Check status: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}
EOF
```

### 5.5 Configure Webhook Service (Generic)

```bash
kubectl apply -n argocd -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.webhook.url: https://webhook.site/your-unique-id
  
  # Or Slack webhook
  service.slack.token: xoxb-YOUR-SLACK-TOKEN
EOF
```

### 5.6 Subscribe Application to Notifications

Subscribe the BankApp to receive notifications:

```bash
# Via CLI annotations
kubectl annotate application bankapp -n argocd \
  notifications.argoproj.io/subscribe.on-sync-succeeded.webhook="" \
  notifications.argoproj.io/subscribe.on-sync-failed.webhook="" \
  notifications.argoproj.io/subscribe.on-health-degraded.webhook="" \
  --overwrite

# For Slack
kubectl annotate application bankapp -n argocd \
  notifications.argoproj.io/subscribe.on-sync-failed.slack="bankapp-alerts" \
  --overwrite
```

Or add to Application manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.webhook: ""
    notifications.argoproj.io/subscribe.on-sync-failed.webhook: ""
    notifications.argoproj.io/subscribe.on-health-degraded.webhook: ""
spec:
  ...
```

### 5.7 View Notification History

```bash
# Check operation state
kubectl get applications bankapp -n argocd -o jsonpath='{.status.operationState}'

# View recent notifications (with jq)
kubectl get events -n argocd --sort-by='.lastTimestamp' | grep bankapp

# Check notification controller logs
kubectl logs -n argocd deployment/argocd-notifications-controller -f
```

### 5.8 Notification Triggers Reference

| Trigger | When | Use Case |
|---------|------|----------|
| `on-sync-succeeded` | Sync completes successfully | Confirm deployment |
| `on-sync-failed` | Sync fails | Alert on deployment error |
| `on-health-degraded` | App health becomes Degraded | Pod crashes, service down |
| `on-health-progressing` | App is healing | Informational (verbose) |
| `on-sync-status-unknown` | Sync status unclear | Edge case alert |

### 5.9 Custom Notification Template

Create custom templates for different teams:

```yaml
template.banking-team-alert: |
  message: |
    🏦 Banking App Alert
    Application: {{.app.metadata.name}}
    Status: {{.app.status.operationState.phase}}
    Timestamp: {{now | date "2006-01-02 15:04:05 MST"}}
    
    ⚠️ Action Required: Contact on-call engineer
    Escalation: banking-team-oncall@company.slack.com
```

---

## 6. ArgoCD Projects and RBAC

### 6.1 The Problem

In production:
- 50+ applications across multiple teams
- Each team should only manage their apps
- Prevent one team from accidentally affecting another's resources
- Different permission levels (dev can deploy, senior can rollback)

**Solution**: ArgoCD **Projects** for access control

### 6.2 Create a Project for BankApp Team

```bash
argocd proj create bankapp-team \
  --description "AI-BankApp team project" \
  --src "https://github.com/<your-username>/AI-BankApp-DevOps.git" \
  --dest "https://kubernetes.default.svc,bankapp" \
  --dest "https://kubernetes.default.svc,monitoring" \
  --orphaned-resources enable
```

**Restrictions Set**:
- Can only source from AI-BankApp repo (not other repos)
- Can only deploy to `bankapp` and `monitoring` namespaces
- Cannot deploy to `kube-system`, `argocd`, or other namespaces
- Orphaned resources (resources removed from Git) are tracked

### 6.3 Move Application to Project

```bash
# Move bankapp to the project
argocd app set bankapp --project bankapp-team

# Verify
argocd app get bankapp | grep project
# Output: project: bankapp-team
```

### 6.4 Test Restrictions

```bash
# Try to add unauthorized destination (should fail)
argocd proj add-destination bankapp-team https://kubernetes.default.svc kube-system 2>&1 || echo "✓ Restricted as expected"

# Try to add unauthorized source repo (should fail)
argocd proj add-source bankapp-team https://github.com/otherteam/app.git 2>&1 || echo "✓ Restricted as expected"
```

### 6.5 RBAC Configuration

Create role-based access control in `argocd-rbac-cm` ConfigMap:

```bash
kubectl apply -n argocd -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # Roles
    p, role:bankapp-dev, applications, get, bankapp-team/*, allow
    p, role:bankapp-dev, applications, sync, bankapp-team/*, allow
    p, role:bankapp-dev, applications, rollback, bankapp-team/*, deny
    p, role:bankapp-dev, repositories, get, *, allow
    
    p, role:bankapp-admin, applications, *, bankapp-team/*, allow
    p, role:bankapp-admin, repositories, *, *, allow
    
    p, role:platform-admin, *, *, *, allow
    
    # Group mappings (from OIDC/OAuth provider)
    g, bankapp-developers, role:bankapp-dev
    g, bankapp-team-leads, role:bankapp-admin
    g, platform-engineering, role:platform-admin
    
    # Individual user mappings
    g, alice@company.com, role:bankapp-admin
    g, bob@company.com, role:bankapp-dev
EOF
```

**Policy Breakdown**:

| Policy | Meaning |
|--------|---------|
| `p, role:bankapp-dev, applications, get, ...` | Can view bankapp-team apps |
| `p, role:bankapp-dev, applications, sync, ...` | Can deploy bankapp-team apps |
| `p, role:bankapp-dev, applications, rollback, ..., deny` | Cannot rollback |
| `p, role:bankapp-admin, applications, *, ...` | Can do anything with apps |
| `p, role:platform-admin, *, *, *, allow` | Super-admin (all actions) |
| `g, bankapp-developers, role:bankapp-dev` | Users in group get role permissions |

### 6.6 RBAC Verification

```bash
# Get current user
kubectl config current-context

# Check what user can do
argocd account can-i sync bankapp

# Check another user (requires token)
argocd account can-i get applications --as bob@company.com
```

### 6.7 Multi-Team Setup

```bash
# Create project for monitoring team
argocd proj create monitoring-team \
  --src "https://github.com/<your-username>/prometheus-configs.git" \
  --dest "https://kubernetes.default.svc,monitoring" \
  --dest "https://kubernetes.default.svc,prometheus"

# Create project for logging team
argocd proj create logging-team \
  --src "https://github.com/<your-username>/elk-configs.git" \
  --dest "https://kubernetes.default.svc,logging"

# Create RBAC for each team
kubectl apply -n argocd -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # BankApp Team
    p, role:bankapp-dev, applications, *, bankapp-team/*, allow
    g, bankapp-developers, role:bankapp-dev
    
    # Monitoring Team
    p, role:monitoring-dev, applications, *, monitoring-team/*, allow
    g, monitoring-developers, role:monitoring-dev
    
    # Logging Team
    p, role:logging-dev, applications, *, logging-team/*, allow
    g, logging-developers, role:logging-dev
    
    # Platform Team (can see everything, manage projects)
    p, role:platform-admin, *, *, *, allow
    g, platform-engineers, role:platform-admin
EOF
```

### 6.8 How Projects Prevent Cross-Team Incidents

**Scenario**: Developer Bob from the Logging team accidentally tries to delete the BankApp

```bash
# Bob tries to sync changes to bankapp (not in logging-team project)
argocd app sync bankapp

# ArgoCD returns error:
# Error: application bankapp is not in any of the allowed projects: [logging-team]
# ✓ Prevented! Bob's permissions don't allow access to bankapp
```

**Result**:
- One team's mistakes don't affect another team
- Clear permission boundaries
- Reduced blast radius of mistakes
- Audit trail shows who tried what

---

## 7. Complete Workflow: Putting It All Together

### 7.1 Day 85 Scenario

**Setup**:
- AI-BankApp with sync waves for ordered deployment
- Manual sync policy for production control
- Monitoring and logging apps via App of Apps pattern
- Notifications to Slack on sync events
- RBAC preventing cross-team access

### 7.2 Step-by-Step Workflow

#### Step 1: Prepare Repository with Sync Waves

```bash
# In your AI-BankApp fork
git checkout -b feat/gitops

# Add sync wave annotations to all manifests
# (see section 2.4 above)

git add k8s/
git commit -m "Add ArgoCD sync wave annotations for ordered deployment"
git push origin feat/gitops
```

#### Step 2: Create App of Apps Structure

```bash
# Create argocd-apps directory
mkdir -p argocd-apps

# Create child applications
cat > argocd-apps/bankapp.yaml << EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: bankapp-team
  source:
    repoURL: https://github.com/<your-username>/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: bankapp
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF

# Create root app
cat > argocd-apps/root-app.yaml << EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: argocd-apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

git add argocd-apps/
git commit -m "Add App of Apps pattern structure"
git push origin feat/gitops
```

#### Step 3: Configure Projects and RBAC

```bash
# Create project
argocd proj create bankapp-team \
  --description "AI-BankApp team" \
  --src "https://github.com/<your-username>/AI-BankApp-DevOps.git" \
  --dest "https://kubernetes.default.svc,bankapp"

# Configure RBAC
kubectl apply -n argocd -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:bankapp-dev, applications, get, bankapp-team/*, allow
    p, role:bankapp-dev, applications, sync, bankapp-team/*, allow
    g, bankapp-developers, role:bankapp-dev
EOF
```

#### Step 4: Deploy Root App

```bash
# Deploy the root application
kubectl apply -f argocd-apps/root-app.yaml

# Wait for child apps to be created
sleep 30

# Verify all apps are synced
argocd app list
```

#### Step 5: Test Sync Waves

```bash
# Watch the ordered deployment
kubectl get pods -n bankapp -w

# In ArgoCD UI: Watch sync progress tab
# Should see resources deployed in wave order:
# Wave -2 (infrastructure) → Wave -1 (config) → Wave 0 (services) → Wave 1 (apps)
```

#### Step 6: Configure Notifications

```bash
# Apply notification config
kubectl apply -n argocd -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-sync-succeeded]
  
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  
  template.app-sync-succeeded: |
    message: "✅ {{.app.metadata.name}} sync succeeded"
  
  template.app-sync-failed: |
    message: "❌ {{.app.metadata.name}} sync FAILED"
  
  service.webhook.url: https://webhook.site/your-id
EOF

# Subscribe app to notifications
kubectl annotate application bankapp -n argocd \
  notifications.argoproj.io/subscribe.on-sync-succeeded.webhook="" \
  --overwrite
```

#### Step 7: Test Manual Sync

```bash
# Change bankapp image (for testing)
# Edit k8s/bankapp-deployment.yml
# Change image tag from :latest to :v1.2.0

git add k8s/
git commit -m "Update bankapp to v1.2.0"
git push origin feat/gitops

# Check status
argocd app get bankapp
# Status should be: OutOfSync (because manual sync is enabled)

# Review changes
argocd app diff bankapp

# Approve and sync
argocd app sync bankapp

# Verify
argocd app get bankapp
# Status should be: Synced
```

#### Step 8: Test Rollback

```bash
# Check history
argocd app history bankapp
# ID  DATE              REVISION
# 1   2026-04-10...     abc1234 (initial)
# 2   2026-04-10...     def5678 (v1.2.0)

# Simulate failure - revert to revision 1
argocd app rollback bankapp 1

# Verify status
argocd app get bankapp
# Status: OutOfSync (cluster at rev 1, HEAD at rev 2)

# Proper fix: git revert
git log --oneline | head -2
git revert HEAD
git push origin feat/gitops

# Wait for ArgoCD to sync
argocd app wait bankapp
```

---

## 8. Key Takeaways

| Concept | Key Point |
|---------|-----------|
| **Sync Strategies** | Automated for dev/staging, Manual for production |
| **Sync Waves** | Control resource deployment order with annotations |
| **Rollbacks** | Use ArgoCD for emergency recovery, git revert for permanent fix |
| **App of Apps** | Manage multiple apps with single parent Application |
| **Notifications** | Alert teams on deployment events |
| **Projects/RBAC** | Restrict teams to their own applications |

---

## 9. Common Issues and Troubleshooting

### Issue: Resources in Waves Not Deploying in Order

**Cause**: Annotations not on correct resource kind (e.g., on Service but not Deployment)

**Fix**: Ensure annotation is on the resource metadata, not spec.template.metadata

```yaml
# ✗ WRONG
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        argocd.argoproj.io/sync-wave: "0"  # In template, doesn't work

# ✓ CORRECT
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "0"  # In metadata, works
spec:
  ...
```

### Issue: Rollback Doesn't Change Git

**Solution**: Always follow ArgoCD rollback with `git revert` to maintain GitOps principles

```bash
argocd app rollback bankapp 3      # Emergency recovery
git revert HEAD && git push        # Make it permanent
```

### Issue: Notifications Not Triggering

**Solution**: Verify ConfigMap and subscription annotations

```bash
# Check config
kubectl get configmap argocd-notifications-cm -n argocd -o yaml

# Check subscription
kubectl get application bankapp -n argocd -o jsonpath='{.metadata.annotations}' | grep notifications

# Check controller logs
kubectl logs -n argocd deployment/argocd-notifications-controller -f
```

### Issue: RBAC Blocking Legitimate Users

**Solution**: Double-check group mappings and OIDC provider configuration

```bash
# Test permissions
argocd account can-i sync bankapp --as user@company.com

# View RBAC config
kubectl get configmap argocd-rbac-cm -n argocd -o yaml
```

---

## 10. Production Readiness Checklist

Before deploying to production:

- [ ] Sync waves configured for all application dependencies
- [ ] Manual sync policy enabled (no automated sync in prod)
- [ ] Notifications configured for sync succeeded/failed events
- [ ] Projects created for each team
- [ ] RBAC policies restricting cross-team access
- [ ] App of Apps pattern implemented for multi-app management
- [ ] Rollback procedure documented and tested
- [ ] git revert workflow established for permanent rollbacks
- [ ] ArgoCD resource quotas and limits configured
- [ ] Backup and disaster recovery plan for ArgoCD data
- [ ] GitOps audit trail enabled (Git history immutable)
- [ ] Production cluster read-only except via ArgoCD

---

## References

- [ArgoCD Official Docs](https://argo-cd.readthedocs.io/)
- [Sync Waves Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [Notifications Guide](https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/)
- [RBAC Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)

---

## Submission Checklist

- [x] Understand sync strategies (automated vs manual)
- [x] Sync waves configuration and ordering
- [x] ArgoCD rollbacks and git revert approach
- [x] App of Apps pattern implementation
- [x] Notifications configuration
- [x] Projects and RBAC setup
- [x] Complete workflows documented
- [x] Troubleshooting guide included
- [x] Production readiness checklist

---

**Next Steps**: Day 86 will cover ArgoCD Security (RBAC deep dive, webhook authentication) and High Availability setup (multi-node ArgoCD deployment).

---

*Last Updated: 2026-04-10*  
*GitOps is not just "deploy from Git" -- it is a complete operational framework for managing production Kubernetes clusters at scale.*
