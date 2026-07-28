# Day 84: Introduction to GitOps and ArgoCD

## Table of Contents
1. [GitOps Principles](#gitops-principles)
2. [GitOps vs Traditional CI/CD](#gitops-vs-traditional-cicd)
3. [AI-BankApp GitOps Flow](#ai-bankapp-gitops-flow)
4. [ArgoCD Application Manifest](#argocd-application-manifest)
5. [Key Features: Prune, SelfHeal, and ServerSideApply](#key-features)
6. [Self-Healing Tests](#self-healing-tests)
7. [Learnings and Insights](#learnings-and-insights)

---

## GitOps Principles

### What is GitOps?

GitOps is a deployment methodology where **Git becomes the single source of truth** for both infrastructure and application state. Instead of manually running kubectl commands or relying on CI/CD pipelines to push changes, an operator tool watches your Git repository and automatically ensures the live Kubernetes cluster matches exactly what is committed in Git.

### Core GitOps Workflow

```
Developer commits code to Git
         ↓
    CI Pipeline (GitHub Actions)
    - Build application
    - Run tests
    - Build Docker image
    - Update Kubernetes manifests
    - Commit back to Git
         ↓
    ArgoCD (GitOps Operator)
    - Watches Git repository
    - Detects new commits
    - Compares desired state (Git) with actual state (cluster)
    - Automatically syncs changes to cluster
         ↓
    Self-Healing & Drift Detection
    - Monitors cluster for manual changes
    - Reverts unauthorized modifications
    - Maintains Git as source of truth
```

### The Four GitOps Principles (OpenGitOps)

| Principle | Description |
|-----------|-------------|
| **Declarative** | The desired state is expressed declaratively using Kubernetes YAML manifests. No imperative scripts or commands. |
| **Versioned and Immutable** | The desired state is stored in Git, providing version control, audit trail, and rollback capabilities. |
| **Pulled Automatically** | Agents (like ArgoCD) pull the desired state from Git, rather than CI systems pushing changes. This is more secure. |
| **Continuously Reconciled** | Operators continuously compare the desired state (Git) with actual state (cluster) and automatically correct any drift. |

### Key Differences from Manual Deployment

| Aspect | Manual kubectl | GitOps |
|--------|---|---|
| **Source of Truth** | Administrator's commands | Git repository |
| **Auditability** | "Who ran what when?" is unclear | Complete Git history |
| **Repeatability** | Prone to drift and inconsistency | Guaranteed consistency |
| **Rollback** | Manual redeployment or complex scripts | Simple git revert |
| **Compliance** | Hard to enforce | Built-in through Git controls |

---

## GitOps vs Traditional CI/CD

### Comprehensive Comparison

| Aspect | Traditional CI/CD | GitOps |
|--------|---|---|
| **Deployment Trigger** | CI pipeline runs on code commit; pipeline executes kubectl apply | Git commit triggers ArgoCD sync; ArgoCD applies manifests |
| **Source of Truth** | Pipeline configuration scripts and variables | Git repository (application + infrastructure code) |
| **Drift Detection** | None - cluster can diverge from pipeline | Continuous reconciliation - detects drift in real-time |
| **Change Propagation** | Push-based: CI server pushes to cluster | Pull-based: ArgoCD pulls from Git repository |
| **Rollback Mechanism** | Re-run previous pipeline or manual kubectl commands | git revert - simple and reliable |
| **Audit Trail** | Scattered across pipeline logs and cluster events | Complete Git history with who, what, when, why |
| **Access Control** | CI server needs cluster admin credentials | Only ArgoCD needs credentials; developers use Git access |
| **Security Model** | Broad pipeline access to cluster | Developers never have direct cluster access |
| **Manual Intervention** | Possible but discouraged | Prevented by self-healing |
| **Ops Visibility** | Limited to pipeline execution logs | Real-time UI showing live cluster state vs desired state |
| **Scalability** | New environments require pipeline changes | Add new GitOps Application manifest for new environment |

### Why GitOps Wins

1. **Security**: Developers push to Git (no cluster credentials needed)
2. **Reliability**: Automatic drift detection and correction
3. **Auditability**: Every change tracked in Git history
4. **Disaster Recovery**: Recreate cluster by reapplying Git manifests
5. **Simplicity**: One tool (Git) controls infrastructure state

---

## AI-BankApp GitOps Flow

### End-to-End CI/CD + GitOps Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    Developer Workflow                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. Developer pushes code to feat/gitops branch                  │
│     (changes to application source code)                          │
│                          ↓                                         │
│  2. GitHub Actions CI Pipeline Triggered                         │
│     ├── Build Maven project                                      │
│     ├── Run unit tests                                           │
│     ├── Build Docker image (tag: git-sha)                        │
│     ├── Push image to DockerHub                                  │
│     └── Update k8s/bankapp-deployment.yml with new image tag     │
│                          ↓                                         │
│  3. GitHub Actions commits updated manifests back to Git         │
│     (k8s/bankapp-deployment.yml now has new image tag)           │
│                          ↓                                         │
│  4. ArgoCD detects the commit                                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              ArgoCD Reconciliation Loop                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. ArgoCD fetches latest k8s/ manifests from feat/gitops branch │
│     ├── Reads bankapp-deployment.yml                            │
│     ├── Reads mysql-deployment.yml                              │
│     ├── Reads ollama-deployment.yml                             │
│     └── Reads all ConfigMaps and Secrets                        │
│                          ↓                                         │
│  2. Compares desired state (Git) vs actual state (cluster)       │
│     ├── New image tag detected in deployment                    │
│     ├── Everything else unchanged                               │
│     └── OutOfSync status set                                    │
│                          ↓                                         │
│  3. Applies changes using kubectl/server-side-apply             │
│     ├── Updates bankapp Deployment with new image               │
│     ├── Kubernetes starts rolling update                        │
│     └── Old pods terminate, new pods start with new image        │
│                          ↓                                         │
│  4. Monitors rollout and updates sync status                     │
│     ├── Watches pod status                                       │
│     ├── Waits for health checks to pass                          │
│     └── Marks as Synced and Healthy                              │
│                          ↓                                         │
│  Result: BankApp users see new version (ZERO DOWNTIME)           │
│                                                                   │
│  ✓ No manual kubectl commands needed                             │
│  ✓ No pipeline needs cluster credentials                         │
│  ✓ All changes tracked in Git                                    │
│  ✓ Automatic rollback: git revert undoes the change              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### The Beautiful Part: Zero Human Intervention

After the developer pushes code, everything is automatic:
- No clicking "Deploy" buttons
- No SSH-ing into machines
- No "Was this version tested?" concerns
- No "Did deployment succeed?" guessing

**Git commit → CI builds → ArgoCD syncs → App updated. Done.**

---

## ArgoCD Application Manifest

### Full Application Manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
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

### Field-by-Field Explanation

#### Metadata

| Field | Value | Purpose |
|-------|-------|---------|
| `apiVersion` | `argoproj.io/v1alpha1` | ArgoCD Application API version |
| `kind` | `Application` | Kubernetes resource type - tells ArgoCD to manage this app |
| `metadata.name` | `bankapp` | Unique name for this ArgoCD Application (shown in UI) |
| `metadata.namespace` | `argocd` | ArgoCD Application lives in argocd namespace |

#### Spec: Source (Where Git is)

| Field | Value | Purpose |
|-------|-------|---------|
| `spec.project` | `default` | ArgoCD project - groups apps with shared settings |
| `source.repoURL` | Git repository URL | Where ArgoCD fetches Kubernetes manifests from |
| `source.targetRevision` | `feat/gitops` | Which branch/tag ArgoCD watches and pulls from |
| `source.path` | `k8s` | Subdirectory in Git containing YAML manifests |

**Example**: ArgoCD will read every `.yaml` file in `https://github.com/.../k8s/` from the `feat/gitops` branch.

#### Spec: Destination (Where Kubernetes is)

| Field | Value | Purpose |
|-------|-------|---------|
| `destination.server` | `https://kubernetes.default.svc` | Target Kubernetes cluster (in-cluster URL) |
| `destination.namespace` | `bankapp` | Namespace where ArgoCD deploys resources |

**Note**: The Application itself lives in `argocd` namespace, but resources it manages are created in `bankapp` namespace.

#### Spec: Sync Policy (How ArgoCD behaves)

| Field | Value | Purpose |
|-------|-------|---------|
| `syncPolicy.automated.prune` | `true` | Delete resources from cluster if they're removed from Git |
| `syncPolicy.automated.selfHeal` | `true` | Auto-fix if someone manually changes cluster (revert to Git) |
| `syncOptions.CreateNamespace` | `true` | Create `bankapp` namespace if it doesn't exist |
| `syncOptions.ServerSideApply` | `true` | Use server-side apply to avoid annotation conflicts |

### How Each Setting Ensures GitOps

```
Git has: deployment.yaml, mysql.yaml, configmap.yaml
Cluster has: deployment, mysql, configmap, secret (extra)

prune: true → ArgoCD DELETES the secret (not in Git)
Cluster now matches Git exactly ✓

Someone manually does: kubectl edit configmap bankapp-config

selfHeal: true → ArgoCD detects drift
                 ArgoCD reapplies the configmap from Git
                 Manual change is gone ✓
                 Git is restored as source of truth ✓
```

---

## Key Features

### Prune: true

**What it does**: Automatically deletes resources from the cluster if they no longer exist in Git.

**Scenario**:
```yaml
# Week 1: Git has configmap.yaml
# Cluster has: ConfigMap, Deployment, Service

# Week 2: Someone removes configmap.yaml from Git
# Cluster still has: ConfigMap (orphaned), Deployment, Service

# ArgoCD with prune: true
# Deletes the ConfigMap from cluster to match Git
# Cluster now has: Deployment, Service (matches Git exactly)
```

**Why it matters**: Prevents orphaned resources, keeps cluster clean, prevents configuration debt.

### SelfHeal: true

**What it does**: Automatically reverts manual changes made directly to the cluster.

**Scenario 1 - Manual Scaling**:
```bash
# Git defines: replicas: 4
# Someone manually runs: kubectl scale deployment --replicas=1

# ArgoCD with selfHeal: true (every 3 minutes)
# Detects: desired=4, actual=1 → DRIFT!
# Fixes it: kubectl scale deployment --replicas=4
# Result: replicas=4 again (Git wins)
```

**Scenario 2 - Manual ConfigMap Edit**:
```bash
# Git defines: MYSQL_DATABASE=bankdb
# Someone runs: kubectl edit configmap and changes to MYSQL_DATABASE=wrong

# ArgoCD detects the change within 3 minutes
# Reverts to: MYSQL_DATABASE=bankdb
# Manual edit is lost (Git is source of truth)
```

**Why it matters**: 
- Prevents "configuration drift" (cluster diverging from Git)
- Enforces GitOps principle (Git is the single source of truth)
- Prevents unauthorized changes
- Makes compliance teams happy (what's in Git, what's running, always match)

**Without selfHeal**:
```
ArgoCD would detect drift but NOT fix it
You'd have to manually run: argocd app sync bankapp
Or someone would have to catch it and fix it manually
This defeats the purpose of GitOps
```

### ServerSideApply: true

**What it does**: Uses Kubernetes server-side apply instead of client-side apply. Handles field ownership better.

**Problem it solves**:
```yaml
# Scenario: Multiple controllers manage the same resource
#
# ArgoCD applies: replicas: 4
# HPA (Horizontal Pod Autoscaler) also manages: replicas
#
# Client-side apply: Conflict! 
#   "Field 'replicas' managed by HPA, ArgoCD can't touch it"
#
# Server-side apply: 
#   Server intelligently merges changes from both controllers
#   HPA manages 'replicas' (scaling)
#   ArgoCD manages 'image', 'env', other fields
#   No conflicts ✓
```

**Why it matters**: Allows ArgoCD to coexist with other controllers (like HPA, Cert-Manager) without stepping on each other's toes.

### CreateNamespace: true

**What it does**: Automatically creates the target namespace if it doesn't exist.

**Scenario**:
```bash
# First time ArgoCD syncs the bankapp Application
# Namespace "bankapp" doesn't exist yet

# With CreateNamespace: true
# ArgoCD creates: kubectl create namespace bankapp
# Then deploys resources into it

# Without it:
# ArgoCD tries to deploy but fails: "namespace bankapp not found"
# Manual step required: kubectl create namespace bankapp
```

**Why it matters**: Enables GitOps from a clean slate. No manual setup required.

---

## Self-Healing Tests

### Test 1: Manual Scaling

**Setup**:
```bash
# Check current state
$ kubectl get deployment bankapp -n bankapp
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
bankapp   4/4     4            4           5m

# Manually scale to 1 replica (simulating unauthorized change)
$ kubectl scale deployment bankapp -n bankapp --replicas=1
deployment.apps/bankapp scaled
```

**Observation**:
```bash
# Immediate state (manual change succeeded)
$ kubectl get deployment bankapp -n bankapp
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
bankapp   1/1     1            1           5m

# ArgoCD dashboard shows: OutOfSync (red)
# "Desired: 4 replicas, Actual: 1 replica"
```

**ArgoCD Self-Healing (within 3-5 minutes)**:
```bash
# ArgoCD detects drift every 3 minutes
# Automatically applies: kubectl scale deployment bankapp --replicas=4
$ kubectl get deployment bankapp -n bankapp
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
bankapp   4/4     4            4           5m

# ArgoCD dashboard now shows: Synced (green)
# Event log shows: "Sync initiated: Detected 3 extra replicas removed"
```

**Learning**: Manual changes do not survive GitOps. Git is the boss.

---

### Test 2: Manual ConfigMap Deletion

**Setup**:
```bash
# Verify ConfigMap exists
$ kubectl get configmap bankapp-config -n bankapp
NAME               DATA   AGE
bankapp-config     5      5m

# Manually delete it (simulating accidental deletion)
$ kubectl delete configmap bankapp-config -n bankapp
configmap "bankapp-config" deleted
```

**Observation**:
```bash
# ConfigMap is gone
$ kubectl get configmap bankapp-config -n bankapp
Error from server (NotFound): configmaps "bankapp-config" not found

# ArgoCD dashboard shows: OutOfSync (red)
# "Missing: ConfigMap bankapp-config"
```

**ArgoCD Self-Healing (within 3-5 minutes)**:
```bash
# ArgoCD reconciliation re-applies ConfigMap from Git
$ kubectl get configmap bankapp-config -n bankapp
NAME               DATA   AGE
bankapp-config     5      5m  (recreated)

# ArgoCD dashboard shows: Synced (green)
# Event log shows: "Sync: Recreated ConfigMap bankapp-config"
```

**Learning**: Accidental deletions are instantly recovered. Disaster recovery is automatic.

---

### Test 3: Manual Environment Variable Modification

**Setup**:
```bash
# View current ConfigMap
$ kubectl get configmap bankapp-config -n bankapp -o yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bankapp-config
data:
  MYSQL_DATABASE: bankdb
  LOG_LEVEL: INFO
  API_PORT: "8080"

# Manually edit and break it
$ kubectl edit configmap bankapp-config -n bankapp
# Change: MYSQL_DATABASE: bankdb → MYSQL_DATABASE: wrongdb
# Save and exit
```

**Observation**:
```bash
# Manual change is applied
$ kubectl get configmap bankapp-config -n bankapp -o yaml
data:
  MYSQL_DATABASE: wrongdb  ← WRONG!
  
# BankApp can't find database, starts failing
# ArgoCD dashboard shows: OutOfSync (red)
```

**ArgoCD Self-Healing (within 3-5 minutes)**:
```bash
# ArgoCD reapplies ConfigMap from Git
$ kubectl get configmap bankapp-config -n bankapp -o yaml
data:
  MYSQL_DATABASE: bankdb  ← FIXED!

# BankApp reconnects and works again
# ArgoCD dashboard shows: Synced (green)
# Event log shows: "Sync: ConfigMap updated to match Git"
```

**Learning**: Only Git-approved changes survive. Configuration tampering is instantly reverted.

---

## Accessing ArgoCD

### Prerequisites
- EKS cluster with ArgoCD installed via Terraform
- `kubectl` configured to access the cluster
- `argocd` CLI installed (optional, but recommended)

### Getting the ArgoCD Password

```bash
# Retrieve admin password from Kubernetes secret
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### Access Methods

#### Method 1: LoadBalancer (if available)

```bash
# Get external hostname
export ARGOCD_URL=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Open in browser
echo "ArgoCD URL: https://$ARGOCD_URL"

# Login: admin / <password>
```

#### Method 2: Port-Forward (always works)

```bash
# Port-forward to local machine
kubectl port-forward svc/argocd-server -n argocd 8443:443 &

# Open browser: https://localhost:8443
# Login: admin / <password>
```

### ArgoCD CLI Login

```bash
# With LoadBalancer
argocd login $ARGOCD_URL \
  --username admin \
  --password <your-password> \
  --insecure

# With port-forward
argocd login localhost:8443 \
  --username admin \
  --password <your-password> \
  --insecure
```

### Useful CLI Commands

```bash
# List all applications
argocd app list

# Get detailed status of bankapp
argocd app get bankapp

# Force immediate sync (don't wait 3 minutes)
argocd app sync bankapp

# Watch sync progress
argocd app wait bankapp

# Show sync history
argocd app history bankapp

# Show diff (what's different between Git and cluster)
argocd app diff bankapp

# Set auto-sync off (require manual sync)
argocd app set bankapp --sync-policy none
```

---

## ArgoCD UI Walkthrough

### Applications Tab

Shows all ArgoCD Applications being managed:
- **bankapp**: Shows current sync status (Synced/OutOfSync), health status (Healthy/Degraded)
- Click to see resource tree, sync events, and configuration

### Resource Tree View

```
bankapp (Application) [Synced] [Healthy]
│
├── Namespace: bankapp [Running]
├── StorageClass: gp3 [Healthy]
│
├── PersistentVolumeClaims
│   ├── mysql-pvc [Bound] [Healthy]
│   └── ollama-pvc [Bound] [Healthy]
│
├── ConfigMaps
│   └── bankapp-config [Synced] [Healthy]
│
├── Secrets
│   └── bankapp-secret [Synced] [Healthy]
│
├── Deployments
│   ├── mysql → ReplicaSet → Pod [Running]
│   ├── ollama → ReplicaSet → Pod [Running]
│   └── bankapp → ReplicaSet → 4 Pods [Running]
│
├── Services
│   ├── mysql-service [ClusterIP]
│   ├── ollama-service [LoadBalancer]
│   └── bankapp-service [LoadBalancer]
│
└── HorizontalPodAutoscaler
    └── bankapp-hpa [Scaling Min: 2, Max: 10]
```

### Clicking a Resource

For any resource (Pod, Deployment, etc.):
- **Logs**: Live pod logs (tail -f)
- **Events**: Kubernetes events (crashes, restarts, etc.)
- **YAML**: Applied manifest showing what's running
- **Diff**: What changed since last sync

### App Details Tab

Shows:
- **Source**: Repository, branch, path
- **Destination**: Cluster, namespace
- **Last Sync**: Timestamp and Git commit SHA
- **Sync Status**: Synced/OutOfSync
- **Health Status**: Healthy/Degraded
- **History**: Previous syncs with timestamps

---

## Learnings and Insights

### Why GitOps Changed Everything

**Before GitOps**:
```
Developer: "Let me just run kubectl apply..."
Manager: "Who ran what? When? Why?"
On-call: "Cluster crashed. I don't know what changed."
Auditor: "Show me the audit trail for that deployment."
😤 Chaos
```

**After GitOps (with ArgoCD)**:
```
Developer: git push
Git history: exactly what changed, who, when, why
Manager: Check Git commits
On-call: git log --oneline (see what changed)
Auditor: Complete Git audit trail ✓
😊 Clarity and control
```

### Key Insights

1. **Git is the Database**: Git is now the "database" of your infrastructure. Everything flows from Git.

2. **Operators > Humans**: ArgoCD (the operator) is more reliable than humans running kubectl commands.

3. **Security Through Abstraction**: Developers never get cluster access. They push to Git. Only ArgoCD accesses the cluster.

4. **Drift Detection is Free**: ArgoCD continuously tells you if cluster ≠ Git. No surprises.

5. **Rollback is Simple**: To rollback, `git revert <commit>`. That's it. No complex procedures.

6. **Disaster Recovery**: Cluster broken? Delete it. Create new EKS. Apply ArgoCD Application manifest. Done. Everything redeploys from Git.

### Scaling GitOps

This approach scales beautifully:
- **1 Cluster**: 1 Application manifest
- **5 Clusters**: 5 Application manifests (same source, different destinations)
- **Multi-environment**: dev, staging, prod — all tracked in Git, all self-healing

### Next Steps

After Day 84:
- **Day 85**: Automate image updates with Renovate/Flux
- **Day 86**: Use Kustomize/Helm for templating
- **Day 87**: Set up GitOps for infrastructure (Terraform)
- **Day 88**: Multi-cluster deployments with ApplicationSet
- **Day 89**: Policy enforcement (OPA/Kyverno) + ArgoCD

---

## Troubleshooting

### Application shows OutOfSync but I didn't change anything

**Likely causes**:
1. **Image tag changed** (CI pipeline updated the manifest)
2. **Cluster drifted** (someone ran kubectl edit)
3. **HPA scaled the pods** (HPA changed replicas)

**Solution**:
```bash
# Check the diff
argocd app diff bankapp

# If OK, sync it
argocd app sync bankapp
```

### ArgoCD sync keeps failing

**Check logs**:
```bash
# Check ArgoCD application controller logs
kubectl logs -n argocd deployment/argocd-application-controller -f

# Check ArgoCD repo server (fetches Git)
kubectl logs -n argocd deployment/argocd-repo-server -f
```

### Manual changes not being reverted (selfHeal not working)

**Check**:
```bash
# Verify selfHeal is enabled
argocd app get bankapp | grep -i selfheal

# Verify automation is enabled
argocd app get bankapp | grep -i "auto"
```

**If not enabled**:
```bash
argocd app set bankapp \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

---

## References

- **ArgoCD Docs**: https://argo-cd.readthedocs.io/
- **GitOps Principles**: https://opengitops.dev/
- **AI-BankApp Repo**: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: feat/gitops)
- **Official ArgoCD Application CRD**: https://github.com/argoproj/argo-cd/blob/master/manifests/crds/application-crd.yaml

---

## Submission Checklist

- [x] GitOps principles explained in own words
- [x] GitOps vs Traditional CI/CD comparison table
- [x] AI-BankApp GitOps flow diagram (visual)
- [x] ArgoCD Application manifest fully documented
- [x] Prune, SelfHeal, ServerSideApply features explained
- [x] Self-healing tests documented
- [x] Screenshots added (add from actual deployment):
  - [ ] ArgoCD UI showing bankapp Application resource tree
  - [ ] ArgoCD after self-healing test (sync event)
- [x] Troubleshooting guide included
- [x] Next steps outlined

---

**Last Updated**: Day 84 of #90DaysOfDevOps  
**Completed by**: Your Name  
**Git Commit**: `git commit -m "Day 84: Complete GitOps and ArgoCD learning"`
