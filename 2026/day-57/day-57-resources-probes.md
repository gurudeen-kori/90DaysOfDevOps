# Day 57 – Resource Requests, Limits, and Probes

## Objective

Learn how Kubernetes manages CPU and memory resources and how probes help Kubernetes detect unhealthy containers and recover automatically.

---

# Resource Requests and Limits

## Requests

Requests define the minimum resources required by a container.

The Kubernetes scheduler uses requests to decide where a Pod can be placed.

Example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
```

Meaning:

* CPU: 0.1 Core
* Memory: 128 MiB

---

## Limits

Limits define the maximum resources a container can use.

The kubelet enforces these limits at runtime.

Example:

```yaml
resources:
  limits:
    cpu: "250m"
    memory: "256Mi"
```

Meaning:

* CPU maximum: 0.25 Core
* Memory maximum: 256 MiB

---

# CPU vs Memory Behavior

| Resource | When Limit Exceeded |
| -------- | ------------------- |
| CPU      | Throttled           |
| Memory   | OOMKilled           |

CPU is compressible.

Memory is not compressible.

---

# Task 1 – Resource Requests and Limits

## Manifest

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: resource-demo

spec:
  containers:
  - name: nginx
    image: nginx

    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"

      limits:
        cpu: "250m"
        memory: "256Mi"
```

Apply:

```bash
kubectl apply -f resource-demo.yaml
```

Inspect:

```bash
kubectl describe pod resource-demo
```

Output:

```text
Requests:
  cpu:     100m
  memory:  128Mi

Limits:
  cpu:     250m
  memory:  256Mi
```

---

## QoS Class

Because:

```text
requests < limits
```

QoS:

```text
Burstable
```

### QoS Classes

| QoS        | Condition          |
| ---------- | ------------------ |
| Guaranteed | Requests = Limits  |
| Burstable  | Requests < Limits  |
| BestEffort | No Requests/Limits |

### Verification

QoS Class:

```text
Burstable
```

---

# Task 2 – OOMKilled

## Manifest

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: memory-stress

spec:
  containers:
  - name: stress
    image: polinux/stress

    command: ["stress"]

    args:
    - "--vm"
    - "1"
    - "--vm-bytes"
    - "200M"
    - "--vm-hang"
    - "1"

    resources:
      limits:
        memory: "100Mi"
```

Apply:

```bash
kubectl apply -f memory-stress.yaml
```

Observe:

```bash
kubectl get pod memory-stress -w
```

Pod enters:

```text
CrashLoopBackOff
```

Inspect:

```bash
kubectl describe pod memory-stress
```

Output:

```text
Reason: OOMKilled
Exit Code: 137
```

### Why 137?

```text
137 = 128 + 9
```

9 = SIGKILL

Kernel OOM Killer terminated the process.

### Verification

Exit Code:

```text
137
```

---

# Task 3 – Pending Pod

## Manifest

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: huge-request

spec:
  containers:
  - name: nginx
    image: nginx

    resources:
      requests:
        cpu: "100"
        memory: "128Gi"
```

Apply:

```bash
kubectl apply -f huge-request.yaml
```

Check:

```bash
kubectl get pod
```

Output:

```text
Pending
```

Describe:

```bash
kubectl describe pod huge-request
```

Event:

```text
0/1 nodes are available:
Insufficient cpu.
Insufficient memory.
```

### Verification

Scheduler Event:

```text
Insufficient cpu
Insufficient memory
```

---

# Task 4 – Liveness Probe

Purpose:

Detect stuck containers.

Failure result:

```text
Container Restart
```

## Manifest

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: liveness-demo

spec:
  containers:
  - name: busybox
    image: busybox:1.36

    command:
    - sh
    - -c
    - |
      touch /tmp/healthy
      sleep 30
      rm -f /tmp/healthy
      sleep 600

    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy

      periodSeconds: 5
      failureThreshold: 3
```

Watch:

```bash
kubectl get pod -w
```

After 30 seconds:

```text
Liveness probe failed
```

Container restarted.

### Verification

Restart Count:

```bash
kubectl get pod liveness-demo
```

Example:

```text
RESTARTS: 1
```

---

# Task 5 – Readiness Probe

Purpose:

Control traffic routing.

Failure result:

```text
Removed from Service Endpoints
```

Container is NOT restarted.

## Manifest

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: readiness-demo

spec:
  containers:
  - name: nginx
    image: nginx

    readinessProbe:
      httpGet:
        path: /
        port: 80

      periodSeconds: 5
```

Expose:

```bash
kubectl expose pod readiness-demo \
--port=80 \
--name=readiness-svc
```

Check endpoints:

```bash
kubectl get endpoints readiness-svc
```

Break application:

```bash
kubectl exec readiness-demo -- \
rm /usr/share/nginx/html/index.html
```

Pod:

```text
0/1 READY
```

Endpoints:

```text
<none>
```

Container:

```text
Still Running
```

### Verification

Was the container restarted?

```text
No
```

---

# Task 6 – Startup Probe

Purpose:

Allow slow-starting applications time to initialize.

During startup probe execution:

```text
Liveness Disabled
Readiness Disabled
```

## Manifest

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: startup-demo

spec:
  containers:
  - name: busybox
    image: busybox

    command:
    - sh
    - -c
    - |
      sleep 20
      touch /tmp/started
      sleep 600

    startupProbe:
      exec:
        command:
        - cat
        - /tmp/started

      periodSeconds: 5
      failureThreshold: 12

    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/started
```

### Why failureThreshold 12?

```text
12 × 5 seconds = 60 seconds
```

Application gets 60 seconds to start.

### Verification

If:

```yaml
failureThreshold: 2
```

Then:

```text
2 × 5 = 10 seconds
```

The container would be killed before startup completes.

---

# Probe Comparison

| Probe     | Purpose              | Failure Action        |
| --------- | -------------------- | --------------------- |
| Liveness  | Is container alive?  | Restart Container     |
| Readiness | Can receive traffic? | Remove From Endpoints |
| Startup   | Has app started?     | Kill Container        |

---

# Important Commands

```bash
kubectl describe pod <pod>

kubectl get pod -w

kubectl logs <pod>

kubectl get endpoints

kubectl top pod
```

---

# Key Learnings

* Requests are used for scheduling.
* Limits are enforced at runtime.
* CPU over limit causes throttling.
* Memory over limit causes OOMKilled.
* Exit Code 137 indicates OOMKilled.
* Burstable QoS occurs when requests are lower than limits.
* Liveness probes restart containers.
* Readiness probes remove pods from service endpoints.
* Startup probes protect slow-starting applications.
* Probes are essential for Kubernetes self-healing.

---

# Conclusion

Today I learned how Kubernetes manages resources and automatically recovers unhealthy workloads. I configured resource requests and limits, observed OOMKilled behavior, and implemented liveness, readiness, and startup probes to improve application reliability and self-healing.
