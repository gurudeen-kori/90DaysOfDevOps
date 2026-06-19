# Day 58 – Metrics Server and Horizontal Pod Autoscaler (HPA)

## Objective

Learn how Kubernetes automatically scales applications based on resource usage by installing the Metrics Server and configuring a Horizontal Pod Autoscaler (HPA).

---

# Task 1: Install the Metrics Server

## What is Metrics Server?

Metrics Server is a cluster-wide aggregator of resource usage data.

It collects CPU and memory metrics from kubelets and exposes them through the Kubernetes Metrics API.

Without Metrics Server:

```bash
kubectl top nodes
kubectl top pods
```

will not work.

HPA also depends on Metrics Server to make scaling decisions.

---

## Installation

For Kind cluster:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Patch for local cluster:

```bash
kubectl patch deployment metrics-server -n kube-system --type=json \
-p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

---

## Verification

```bash
kubectl get pods -n kube-system | grep metrics-server
```

```bash
kubectl top nodes
```

```bash
kubectl top pods -A
```

### Observation

Metrics API became available after Metrics Server was healthy and the APIService status changed to:

```bash
v1beta1.metrics.k8s.io   True
```

---

# Task 2: Explore kubectl top

## Commands

```bash
kubectl top nodes
```

```bash
kubectl top pods -A
```

```bash
kubectl top pods -A --sort-by=cpu
```

---

## Difference Between Usage and Requests

### kubectl top

Shows actual resource consumption.

Example:

```text
CPU: 20m
Memory: 30Mi
```

### kubectl describe pod

Shows configured requests and limits.

Example:

```yaml
resources:
  requests:
    cpu: 200m
  limits:
    cpu: 500m
```

Actual usage and configured requests are different values.

---

# Task 3: Create a Deployment with CPU Requests

## Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache

  template:
    metadata:
      labels:
        app: php-apache

    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example

        resources:
          requests:
            cpu: 200m

        ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Expose:

```bash
kubectl expose deployment php-apache --port=80
```

---

## Why CPU Requests Are Required

HPA calculates utilization using:

```text
CPU Utilization =
Current CPU Usage / CPU Request × 100
```

Without CPU requests HPA cannot calculate utilization.

Common error:

```text
missing request for cpu
```

---

# Task 4: Create HPA Imperatively

## Create HPA

```bash
kubectl autoscale deployment php-apache \
--cpu-percent=50 \
--min=1 \
--max=10
```

Verify:

```bash
kubectl get hpa
```

```bash
kubectl describe hpa php-apache
```

---

## HPA Result

Example:

```text
TARGETS
cpu: 0%/50%
```

Meaning:

* Current utilization = 0%
* Target utilization = 50%

Since current utilization is below target, scaling does not occur.

---

# Task 5: Generate Load and Watch Autoscaling

## Load Generator

```bash
kubectl run load-generator \
--image=busybox:1.36 \
--restart=Never \
-- /bin/sh -c \
"while true; do wget -q -O- http://php-apache; done"
```

Watch HPA:

```bash
kubectl get hpa php-apache --watch
```

Watch Deployment:

```bash
kubectl get deployment php-apache -w
```

---

## Expected Behavior

CPU utilization rises.

Example:

```text
cpu: 120%/50%
```

HPA calculates desired replicas and increases pod count.

Example:

```text
1 → 2 → 3 → 4
```

After deleting load-generator:

```bash
kubectl delete pod load-generator
```

Scale-down begins after stabilization period.

---

# How HPA Calculates Desired Replicas

Formula:

```text
desiredReplicas =
ceil(
currentReplicas *
(currentUsage / targetUsage)
)
```

Example:

```text
Current Replicas = 2
Current CPU = 100%
Target CPU = 50%

desiredReplicas =
ceil(2 × 100/50)

desiredReplicas = 4
```

---

# Task 6: Declarative HPA (autoscaling/v2)

## HPA Manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: php-apache

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache

  minReplicas: 1
  maxReplicas: 4

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 2
        periodSeconds: 15

    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
```

Apply:

```bash
kubectl apply -f hpa.yaml
```

---

# What Does the Behavior Section Control?

The behavior section controls scaling speed.

## Scale Up

```yaml
scaleUp:
```

Controls how quickly new pods are added.

## Scale Down

```yaml
scaleDown:
```

Controls how quickly pods are removed.

Example:

```yaml
stabilizationWindowSeconds: 300
```

Waits 5 minutes before scaling down.

This prevents scaling flaps.

---

# autoscaling/v1 vs autoscaling/v2

| Feature          | v1      | v2          |
| ---------------- | ------- | ----------- |
| CPU Metrics      | Yes     | Yes         |
| Memory Metrics   | No      | Yes         |
| Custom Metrics   | No      | Yes         |
| External Metrics | No      | Yes         |
| Scaling Behavior | No      | Yes         |
| Production Usage | Limited | Recommended |

---

# Screenshots

Add screenshots for:

1. kubectl top nodes
2. kubectl top pods -A
3. kubectl get hpa
4. kubectl describe hpa
5. HPA scaling events
6. Deployment scaling from 1 to multiple replicas

---

# Task 7: Cleanup

```bash
kubectl delete hpa php-apache
```

```bash
kubectl delete svc php-apache
```

```bash
kubectl delete deployment php-apache
```

```bash
kubectl delete pod load-generator
```

Keep Metrics Server installed for future HPA exercises.

---

# Key Learnings

* Metrics Server provides CPU and memory usage data.
* HPA requires CPU requests to calculate utilization.
* kubectl top shows actual usage, not requests or limits.
* HPA scales applications automatically based on metrics.
* autoscaling/v2 supports behavior policies and multiple metrics.
* Scale-up is fast; scale-down is intentionally slower.
* HPA is commonly used in production to handle varying traffic loads.
