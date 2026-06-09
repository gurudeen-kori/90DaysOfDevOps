# Day 52 – Kubernetes Namespaces and Deployments

## Objective

Today I learned how Kubernetes Namespaces help organize and isolate resources within a cluster and how Deployments provide self-healing, scaling, and rolling update capabilities for applications.

---

# What are Namespaces?

Namespaces are logical partitions inside a Kubernetes cluster. They help separate environments, teams, or applications without requiring multiple clusters.

### Why Use Namespaces?

* Resource isolation
* Environment separation (Dev, Staging, Production)
* Access control using RBAC
* Better resource management
* Avoid naming conflicts

### Default Kubernetes Namespaces

```bash
kubectl get namespaces
```

Built-in namespaces:

| Namespace       | Purpose                         |
| --------------- | ------------------------------- |
| default         | Default workspace for resources |
| kube-system     | Kubernetes system components    |
| kube-public     | Publicly readable resources     |
| kube-node-lease | Node heartbeat tracking         |

To inspect system components:

```bash
kubectl get pods -n kube-system
```

---

# Creating Custom Namespaces

Created development and staging namespaces:

```bash
kubectl create namespace dev
kubectl create namespace staging
```

Created production namespace using a manifest:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

Apply:

```bash
kubectl apply -f namespace.yaml
```

Verify:

```bash
kubectl get namespaces
```

---

# Running Pods in Different Namespaces

Development Pod:

```bash
kubectl run nginx-dev --image=nginx:latest -n dev
```

Staging Pod:

```bash
kubectl run nginx-staging --image=nginx:latest -n staging
```

View all Pods:

```bash
kubectl get pods -A
```

---

# Creating a Deployment

## Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
  labels:
    app: nginx

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
      - name: nginx
        image: nginx:1.24

        ports:
        - containerPort: 80
```

Apply Deployment:

```bash
kubectl apply -f nginx-deployment.yaml
```

Verify:

```bash
kubectl get deployments -n dev
kubectl get pods -n dev
```

---

# Deployment Manifest Explanation

### metadata

Stores object name, namespace, and labels.

### replicas

Defines the desired number of Pod replicas.

```yaml
replicas: 3
```

### selector

Deployment uses labels to identify and manage Pods.

```yaml
selector:
  matchLabels:
    app: nginx
```

### template

Blueprint used to create Pods.

### containers

Defines the container image and configuration.

---

# Understanding Deployment Status

```bash
kubectl get deployments -n dev
```

### READY

Number of available replicas versus desired replicas.

Example:

```text
3/3
```

### UP-TO-DATE

Pods updated to the latest deployment specification.

### AVAILABLE

Pods available and serving traffic.

---

# Self-Healing Demonstration

List Pods:

```bash
kubectl get pods -n dev
```

Delete one Pod:

```bash
kubectl delete pod <pod-name> -n dev
```

Immediately check again:

```bash
kubectl get pods -n dev
```

### Observation

A new Pod was automatically created because the Deployment always maintains the desired state.

### Standalone Pod vs Deployment

| Standalone Pod      | Deployment                       |
| ------------------- | -------------------------------- |
| Deleted permanently | Recreated automatically          |
| No controller       | Managed by Deployment controller |
| Not self-healing    | Self-healing                     |

The replacement Pod receives a different name because it is newly created.

---

# Scaling Deployments

## Imperative Scaling

Scale up:

```bash
kubectl scale deployment nginx-deployment --replicas=5 -n dev
```

Scale down:

```bash
kubectl scale deployment nginx-deployment --replicas=2 -n dev
```

### Observation

When scaling down, Kubernetes terminates extra Pods until the desired replica count is reached.

---

## Declarative Scaling

Modify the Deployment manifest:

```yaml
replicas: 4
```

Apply:

```bash
kubectl apply -f nginx-deployment.yaml
```

Kubernetes reconciles the cluster state to match the manifest.

---

# Rolling Updates

Update the container image:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev
```

Monitor rollout:

```bash
kubectl rollout status deployment/nginx-deployment -n dev
```

### What Happens?

* New Pods are created gradually.
* Old Pods are removed one by one.
* Application remains available.
* Zero downtime deployment.

Check rollout history:

```bash
kubectl rollout history deployment/nginx-deployment -n dev
```

---

# Rollback

Rollback to the previous version:

```bash
kubectl rollout undo deployment/nginx-deployment -n dev
```

Verify:

```bash
kubectl describe deployment nginx-deployment -n dev | grep Image
```

### Observation

The Deployment reverted from:

```text
nginx:1.25
```

back to:

```text
nginx:1.24
```

---

# Cleanup

Delete resources:

```bash
kubectl delete deployment nginx-deployment -n dev

kubectl delete pod nginx-dev -n dev

kubectl delete pod nginx-staging -n staging

kubectl delete namespace dev staging production
```

Verify cleanup:

```bash
kubectl get namespaces
kubectl get pods -A
```

All created resources were successfully removed.

---

# Key Learnings

* Namespaces logically isolate resources within a cluster.
* Deployments maintain the desired state of applications.
* Deployments provide self-healing capabilities.
* Scaling can be done imperatively or declaratively.
* Rolling updates enable zero-downtime deployments.
* Rollbacks allow quick recovery from failed releases.
* Deployments use ReplicaSets to manage Pods.

## Commands Practiced

```bash
kubectl get namespaces
kubectl get pods -A
kubectl create namespace
kubectl apply -f
kubectl get deployments
kubectl scale deployment
kubectl set image
kubectl rollout status
kubectl rollout history
kubectl rollout undo
kubectl delete
```

#90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham
