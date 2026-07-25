# Day 82: EKS Networking with Gateway API and Persistent Storage

## Overview
Today I moved beyond raw manifests to production-grade networking for the AI-BankApp: the Kubernetes Gateway API with Envoy Gateway, automated TLS via cert-manager, and a deep look at how EBS-backed persistent storage actually behaves on EKS.

---

## Table of Contents
1. [Gateway API vs Ingress](#gateway-api-vs-ingress)
2. [Gateway API Architecture](#gateway-api-architecture)
3. [Installing Envoy Gateway](#installing-envoy-gateway)
4. [Gateway API Resources Explained](#gateway-api-resources-explained)
5. [Session Affinity for AI-BankApp](#session-affinity-for-ai-bankapp)
6. [TLS with cert-manager](#tls-with-cert-manager)
7. [EBS Persistent Storage](#ebs-persistent-storage)
8. [HPA and Resource Budget](#hpa-and-resource-budget)
9. [Verification & Screenshots](#verification--screenshots)
10. [Clean Up](#clean-up)

---

## Gateway API vs Ingress

| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| **API maturity** | Stable but limited | GA since Kubernetes 1.26 |
| **Traffic splitting** | Not supported | Built-in (weighted backends) |
| **Header matching** | Annotation-dependent | Native HTTPRoute rules |
| **Role separation** | Single resource | GatewayClass (infra) → Gateway (ops) → HTTPRoute (dev) |
| **TLS management** | Annotation-based | Native TLS config in Gateway listeners |
| **Session affinity** | Not standardized | BackendTrafficPolicy (with Envoy) |

**Why this matters for AI-BankApp:** The app uses Spring Security with form-based login (stateful sessions), and the team wants canary-style rollouts for the Ollama-integrated features later. Ingress cannot express weighted backends or native cookie affinity — Gateway API can.

---

## Gateway API Architecture

```
[Internet]
    |
[AWS NLB] (auto-created by Envoy Gateway when a Gateway resource is applied)
    |
[Gateway: bankapp-gateway]  (namespace: bankapp)
  |-- Listener: HTTP  (port 80)
  |-- Listener: HTTPS (port 443, TLS terminated using bankapp-tls Secret)
    |
[HTTPRoute: bankapp-route]
  |-- parentRefs: bankapp-gateway (sectionName: http, https)
  |-- rule: PathPrefix "/" -> bankapp-service:8080
    |
[BackendTrafficPolicy: bankapp-session]
  |-- targetRef: HTTPRoute bankapp-route
  |-- ConsistentHash load balancing on Cookie: BANKAPP_AFFINITY
    |
[Service: bankapp-service:8080] (ClusterIP)
    |
[Pods: bankapp x2-4]  (scaled by HPA, session-sticky via cookie)
```

**Flow explanation:**
1. A client resolves `<NLB-IP>.nip.io` and connects to the AWS NLB.
2. The NLB forwards to the Envoy Gateway data plane, which is bound to the `Gateway` resource's listeners (80/443).
3. The `HTTPRoute` matches the path and forwards to `bankapp-service`.
4. The `BackendTrafficPolicy` ensures repeat requests from the same browser (same cookie) land on the same pod.
5. The `Service` load-balances (respecting affinity) across the BankApp pod replicas.

---

## Installing Envoy Gateway

### Step 1: Install Envoy Gateway via Helm

```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version v1.4.0 \
  -n envoy-gateway-system --create-namespace \
  --wait
```

### Step 2: Verify Installation

```bash
kubectl get pods -n envoy-gateway-system
kubectl get gatewayclass
```

**Expected output:**
```
NAME             READY   STATUS    RESTARTS   AGE
envoy-gateway-x  1/1     Running   0          2m

NAME             CONTROLLER                                     ACCEPTED
envoy-gateway    gateway.envoyproxy.io/gatewayclass-controller   True
```

### Step 3: Install Gateway API CRDs (if not already present)

```bash
kubectl get crd gateways.gateway.networking.k8s.io 2>/dev/null || \
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

**What this installs:**
- `GatewayClass`
- `Gateway`
- `HTTPRoute`
- `ReferenceGrant`
- (Envoy adds `BackendTrafficPolicy`, `ClientTrafficPolicy`, etc. as extended CRDs)

---

## Gateway API Resources Explained

### 1. GatewayClass — "Which controller handles Gateways"

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

- **Owned by:** Infrastructure/platform team.
- **Purpose:** Declares that any `Gateway` referencing `envoy-gateway` will be implemented by the Envoy Gateway controller.
- **Analogy:** "Which vendor builds the front door" — decided once, used by every Gateway.

### 2. Gateway — "The actual load balancer + listeners"

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bankapp-gateway
  namespace: bankapp
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      hostname: <your-ip>.nip.io
      tls:
        mode: Terminate
        certificateRefs:
          - name: bankapp-tls
```

- **Owned by:** Operations team.
- **Purpose:** When applied, Envoy Gateway provisions an **AWS Network Load Balancer** automatically and configures it to listen on 80 and 443.
- **TLS termination happens here** — the Gateway decrypts HTTPS traffic using the `bankapp-tls` Secret before forwarding to the backend as plain HTTP internally.

### 3. HTTPRoute — "Where traffic goes"

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bankapp-route
  namespace: bankapp
spec:
  parentRefs:
    - name: bankapp-gateway
      sectionName: https
    - name: bankapp-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: bankapp-service
          port: 8080
```

- **Owned by:** Development team.
- **Purpose:** Maps URL paths (and optionally headers/hosts) to backend Services. Attaches to **both** the HTTP and HTTPS listeners via `sectionName`.
- **Key advantage over Ingress:** could add `weight:` on multiple `backendRefs` for canary traffic splitting with zero extra tooling.

### 4. BackendTrafficPolicy — "How traffic to the backend behaves"

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: bankapp-session
  namespace: bankapp
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: bankapp-route
  loadBalancer:
    type: ConsistentHash
    consistentHash:
      type: Cookie
      cookie:
        name: BANKAPP_AFFINITY
        ttl: 3600s
```

- **Owned by:** Development/Platform team.
- **Purpose:** Envoy-specific extension CRD that configures load-balancing behavior, retries, timeouts, and — critically for this app — **cookie-based session affinity**.
- Not part of the core Gateway API spec; this is how Envoy Gateway implements what other controllers (like Istio) would configure differently.

---

## Session Affinity for AI-BankApp

### Why is this needed?

The AI-BankApp uses **Spring Security with form-based login**, which stores session state **in memory on the pod** that handled the login request (no distributed session store like Redis is configured).

```
Without session affinity:
  User logs in  → hits Pod A → session created on Pod A
  Next request  → Service load-balances → hits Pod B
  Pod B has no idea who this user is → user is logged out ❌

With BANKAPP_AFFINITY cookie:
  User logs in  → hits Pod A → session created on Pod A
                → Envoy sets BANKAPP_AFFINITY cookie tied to Pod A
  Next request  → cookie present → Envoy consistently routes to Pod A ✅
  User stays logged in for the full session (ttl: 3600s)
```

### How ConsistentHash works

Envoy Gateway uses a **consistent hashing** algorithm keyed on the cookie value. This means:
- The same cookie value always maps to the same backend pod (as long as the pod set doesn't change drastically).
- If a pod is removed (scale-down or crash), only sessions hashed to *that* pod are redistributed — not all sessions.
- The `ttl: 3600s` matches Spring Security's session timeout, so the cookie expires in step with the app-level session.

---

## TLS with cert-manager

### Step 1: Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set crds.enabled=true \
  --wait
```

### Step 2: Verify

```bash
kubectl get pods -n cert-manager
```

**Expected output:**
```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-7d...                         1/1     Running   0          1m
cert-manager-cainjector-...                1/1     Running   0          1m
cert-manager-webhook-...                   1/1     Running   0          1m
```

### Step 3: ClusterIssuer (Let's Encrypt, Gateway API HTTP-01)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - group: gateway.networking.k8s.io
                kind: Gateway
                name: bankapp-gateway
                namespace: bankapp
```

### How the automated TLS flow works

```
1. A Certificate resource (or annotation) requests a cert for the Gateway
2. cert-manager talks to Let's Encrypt (ACME) and requests an HTTP-01 challenge
3. Let's Encrypt gives cert-manager a token to publish at:
     http://<hostname>/.well-known/acme-challenge/<token>
4. cert-manager creates a TEMPORARY HTTPRoute on the bankapp-gateway
   that answers this exact path with the expected response
5. Let's Encrypt's servers hit that URL over the real NLB and verify it
6. Once verified, Let's Encrypt issues the certificate
7. cert-manager stores it in the "bankapp-tls" Kubernetes Secret
8. The Gateway's HTTPS listener (tls.certificateRefs) references
   that Secret, and Envoy starts terminating TLS with the real cert
9. cert-manager auto-renews before the 90-day Let's Encrypt cert expires
```

### DNS for testing: nip.io

Since a real domain isn't required for lab testing, `nip.io` is used — a wildcard DNS service where `1.2.3.4.nip.io` resolves directly to `1.2.3.4`.

```bash
export GATEWAY_IP=$(kubectl get gateway bankapp-gateway -n bankapp -o jsonpath='{.status.addresses[0].value}')
export HOSTNAME="${GATEWAY_IP}.nip.io"
echo "HTTPS URL: https://$HOSTNAME"
```

---

## EBS Persistent Storage

### Storage Flow

```
StorageClass (gp3)
      |
      v
PersistentVolumeClaim (mysql-pvc / ollama-pvc)
      |  (WaitForFirstConsumer binding mode)
      v
Pod is scheduled to a node in AZ "X"
      |
      v
PersistentVolume dynamically provisioned
      |
      v
EBS Volume created in AZ "X" (matches the pod's node)
      |
      v
Volume attached to the node, mounted into the Pod
```

### Check the Storage Setup

```bash
# StorageClass
kubectl get storageclass gp3

# PVCs
kubectl get pvc -n bankapp

# PVs (dynamically provisioned)
kubectl get pv
```

**Expected output:**
```
NAME                      STATUS   VOLUME         CAPACITY   STORAGECLASS
mysql-pvc                 Bound    pvc-abc123...  5Gi        gp3
ollama-pvc                Bound    pvc-def456...  10Gi       gp3
```

### Find the Actual EBS Volumes in AWS

```bash
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/created-by,Values=ebs.csi.aws.com" \
  --query "Volumes[*].{ID:VolumeId,Size:Size,AZ:AvailabilityZone,State:State}" \
  --output table \
  --region ap-south-1
```

### Key EBS Concepts on EKS

| Concept | Explanation |
|---------|-------------|
| **WaitForFirstConsumer** | The volume is only created once a pod claims it — and in the *same AZ* as that pod. Prevents cross-AZ mismatches. |
| **ReadWriteOnce (RWO)** | EBS can attach to only one node at a time. This is why MySQL and Ollama both use `Recreate` deployment strategy — the old pod must fully terminate (releasing the volume) before a new pod can attach it. |
| **gp3** | Latest-generation SSD; 3,000 IOPS baseline included, cheaper per-GB than gp2, and supports independent IOPS/throughput tuning. |
| **allowVolumeExpansion: true** | Volumes can be resized (grown) without deleting and recreating the PVC. |

### Testing Persistence

```bash
# Check current MySQL data
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"

# Delete the pod
kubectl delete pod -n bankapp -l app=mysql

# Watch it recreate
kubectl get pods -n bankapp -l app=mysql -w

# Verify data survived
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"
```

**Result:** The database list is identical before and after pod deletion — because the EBS volume is a separate AWS resource from the pod's lifecycle. Kubernetes just re-attaches the same volume to the new pod.

---

## HPA and Resource Budget

### Check HPA

```bash
kubectl get hpa -n bankapp
```

**Expected output:**
```
NAME          REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS
bankapp-hpa   Deployment/bankapp    15%/70%   2         4         2
```

### Check Resource Usage

```bash
kubectl top nodes
kubectl top pods -n bankapp
```

### Resource Budget Table (3x t3.medium nodes)

| Component | CPU Request | Memory Request | Instances |
|-----------|-------------|-----------------|-----------|
| **BankApp** | 250m | 256Mi | 2–4 pods |
| **MySQL** | 250m | 256Mi | 1 pod |
| **Ollama** | 900m | 2Gi | 1 pod |
| **Init containers** | 50m | 32Mi | temporary |
| **System pods** | ~500m | ~500Mi | per node |
| **Total available** | 6000m (3 nodes) | 12Gi (3 nodes) | — |

**Observation:** Ollama is the heaviest single consumer (900m CPU, 2Gi memory). If BankApp scales to its max of 4 pods, total CPU requests reach roughly:
```
BankApp: 4 x 250m = 1000m
MySQL:       250m
Ollama:      900m
System:    ~1500m (3 nodes x 500m)
-----------------------------
Total:     ~3650m out of 6000m available (~61%)
```
This leaves comfortable headroom, but confirms Ollama is the resource to watch when planning node sizing or considering a move to Fargate/dedicated node group for AI workloads.

---

## Verification & Screenshots

### Gateway Status (NLB Address)

```bash
kubectl get gateway -n bankapp -w
```

**Expected output:**
```
NAME               CLASS           ADDRESS                                              PROGRAMMED   AGE
bankapp-gateway    envoy-gateway   a1b2c3d4e5f6.elb.ap-south-1.amazonaws.com            True         3m
```

```bash
export GATEWAY_IP=$(kubectl get gateway bankapp-gateway -n bankapp -o jsonpath='{.status.addresses[0].value}')
echo "App URL: http://$GATEWAY_IP"
```

### Test Access

```bash
curl http://$GATEWAY_IP
```

### PVC Bound to EBS

```bash
kubectl get pvc -n bankapp
```

**Expected output:**
```
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
mysql-pvc    Bound    pvc-5b91db3a-5c8d-4932-805a-b8fb0acc3b36   5Gi        RWO            gp3
ollama-pvc   Bound    pvc-7c8e24af-56ea-4b98-db4c-c95eb1f6768e   10Gi       RWO            gp3
```

### EKS Console Verification

```
AWS Console → EC2 → Load Balancers
  → An NLB named similar to "k8s-bankapp-bankappg-xxxx" appears automatically,
    created by the Envoy Gateway controller when the Gateway resource was applied.

AWS Console → EC2 → Volumes
  → Two gp3 volumes (5Gi, 10Gi) tagged with kubernetes.io/created-by = ebs.csi.aws.com
```

---

## Clean Up

Keep the cluster running for Day 83, but remove the workload to save cost:

```bash
kubectl delete -f k8s/gateway.yml 2>/dev/null
kubectl delete -f k8s/hpa.yml
kubectl delete -f k8s/bankapp-deployment.yml
kubectl delete -f k8s/ollama-deployment.yml
kubectl delete -f k8s/mysql-deployment.yml
kubectl delete -f k8s/service.yml
kubectl delete -f k8s/secrets.yml
kubectl delete -f k8s/configmap.yml
kubectl delete -f k8s/pvc.yml
kubectl delete -f k8s/pv.yml
kubectl delete -f k8s/namespace.yml
```

**Note:** Deleting the Gateway also removes the AWS NLB it created — verify in the AWS Console that the load balancer is gone to avoid lingering charges.

---

## Key Learnings

1. **Gateway API cleanly separates roles** — infra (GatewayClass), ops (Gateway), and dev (HTTPRoute) no longer fight over one Ingress resource.
2. **The Gateway resource directly drives cloud infrastructure** — applying it created a real AWS NLB with zero manual console clicks.
3. **Session affinity is a real production requirement**, not a nice-to-have, for any app using in-memory sessions (like Spring Security here). `BackendTrafficPolicy` with `ConsistentHash` + Cookie solves this cleanly at the traffic layer instead of requiring app-level changes (e.g., sticky sessions or an external session store).
4. **cert-manager + Gateway API automates the entire ACME HTTP-01 flow**, including the tricky part of temporarily exposing a challenge response — no manual certificate handling needed.
5. **EBS volumes are AZ-locked.** `WaitForFirstConsumer` is what prevents Kubernetes from provisioning a volume in the wrong AZ relative to where the pod actually lands.
6. **RWO + Recreate strategy is intentional, not a bug** — EBS volumes physically cannot attach to two nodes at once, so MySQL/Ollama pods must fully terminate before their replacement can mount the same volume.
7. **Ollama dominates the resource budget** — any future node-sizing decisions should treat it as the primary constraint.

---

## References

- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway Documentation](https://gateway.envoyproxy.io/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [AWS EBS CSI Driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- [AI-BankApp GitHub (feat/gitops)](https://github.com/TrainWithShubham/AI-BankApp-DevOps)

---

**Date:** July 25, 2026
**Cluster:** bankapp-eks (ap-south-1)
**Status:** ✅ Gateway API + cert-manager + EBS storage verified
