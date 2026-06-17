# Day 56 – Kubernetes StatefulSets

## Objective

Learn how StatefulSets provide stable identities, ordered deployment, and persistent storage for stateful applications such as MySQL, PostgreSQL, MongoDB, Redis, and Kafka.

---

# What is a StatefulSet?

A StatefulSet is a Kubernetes workload resource designed for applications that require:

* Stable pod names
* Stable network identities
* Persistent storage per replica
* Ordered pod creation and deletion

Unlike Deployments, StatefulSets maintain the identity of each pod even after restarts.

---

# Deployment vs StatefulSet

| Feature        | Deployment         | StatefulSet           |
| -------------- | ------------------ | --------------------- |
| Pod Names      | Random             | Stable and Ordered    |
| Startup Order  | Parallel           | Sequential            |
| Shutdown Order | Random             | Reverse Order         |
| Storage        | Shared/External    | Dedicated PVC per Pod |
| DNS Identity   | No Stable Identity | Stable DNS Name       |
| Best For       | Stateless Apps     | Stateful Apps         |

### Examples

**Deployment**

* Frontend Applications
* APIs
* Nginx
* Microservices

**StatefulSet**

* MySQL
* PostgreSQL
* MongoDB
* Redis
* Kafka

---

# Task 1: Deployment Pod Names

Created a Deployment with 3 replicas.

```bash
kubectl get pods
```

Example:

```text
nginx-deployment-6f7c4b9f8c-abcde
nginx-deployment-6f7c4b9f8c-fghij
nginx-deployment-6f7c4b9f8c-klmno
```

Deleted one pod:

```bash
kubectl delete pod nginx-deployment-6f7c4b9f8c-abcde
```

New pod received a different random name.

### Why is this a problem?

Database clusters need fixed identities.

Example:

* mysql-0 = Primary
* mysql-1 = Replica
* mysql-2 = Replica

Random names would break cluster communication.

---

# Task 2: Create a Headless Service

## Manifest

```yaml
apiVersion: v1
kind: Service

metadata:
  name: stateful-service

spec:
  clusterIP: None

  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f headless-service.yaml
```

Verify:

```bash
kubectl get svc
```

Output:

```text
NAME               TYPE        CLUSTER-IP
stateful-service   ClusterIP   None
```

### Observation

CLUSTER-IP shows:

```text
None
```

This confirms it is a Headless Service.

---

# Task 3: Create a StatefulSet

## Manifest

```yaml
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: nginx

spec:
  serviceName: stateful-service
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
        image: nginx

        ports:
        - containerPort: 80

        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html

  volumeClaimTemplates:
  - metadata:
      name: web-data

    spec:
      accessModes:
      - ReadWriteOnce

      storageClassName: standard

      resources:
        requests:
          storage: 100Mi
```

Apply:

```bash
kubectl apply -f statefulset.yaml
```

Watch creation:

```bash
kubectl get pods -w
```

Observed order:

```text
nginx-0
nginx-1
nginx-2
```

Pods were created sequentially.

---

# PVC Creation

Check PVCs:

```bash
kubectl get pvc
```

Output:

```text
web-data-nginx-0
web-data-nginx-1
web-data-nginx-2
```

Each pod received its own PVC.

---

# Task 4: Stable Network Identity

DNS format:

```text
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

Verification:

```bash
kubectl run dns-test \
--image=busybox:1.36 \
--restart=Never \
-it --rm \
-- nslookup nginx-0.stateful-service.default.svc.cluster.local
```

Output:

```text
Name:
nginx-0.stateful-service.default.svc.cluster.local

Address:
10.244.2.10
```

Compare with:

```bash
kubectl get pods -o wide
```

Output:

```text
nginx-0  10.244.2.10
```

### Result

DNS IP matched Pod IP.

Repeated for:

```text
nginx-1.stateful-service.default.svc.cluster.local
nginx-2.stateful-service.default.svc.cluster.local
```

Both resolved successfully.

---

# Task 5: Persistent Storage Verification

Write unique data:

```bash
kubectl exec nginx-0 -- \
sh -c "echo 'Data from nginx-0' > /usr/share/nginx/html/index.html"
```

Verify:

```bash
kubectl exec nginx-0 -- cat /usr/share/nginx/html/index.html
```

Output:

```text
Data from nginx-0
```

Delete pod:

```bash
kubectl delete pod nginx-0
```

Wait for recreation.

Verify again:

```bash
kubectl exec nginx-0 -- cat /usr/share/nginx/html/index.html
```

Output:

```text
Data from nginx-0
```

### Result

Data survived pod recreation because the pod reused the same PVC.

---

# Task 6: Ordered Scaling

Scale Up

```bash
kubectl scale sts nginx --replicas=5
```

Creation order:

```text
nginx-3
nginx-4
```

Scale Down

```bash
kubectl scale sts nginx --replicas=3
```

Deletion order:

```text
nginx-4
nginx-3
```

Reverse order termination confirmed.

Check PVCs:

```bash
kubectl get pvc
```

Output:

```text
web-data-nginx-0
web-data-nginx-1
web-data-nginx-2
web-data-nginx-3
web-data-nginx-4
```

### Observation

All PVCs remained even after scale down.

PVC Count:

```text
5
```

---

# Task 7: Cleanup

Delete StatefulSet:

```bash
kubectl delete sts nginx
```

Delete Service:

```bash
kubectl delete svc stateful-service
```

Check PVCs:

```bash
kubectl get pvc
```

PVCs still existed.

Delete manually:

```bash
kubectl delete pvc --all
```

### Observation

PVCs were NOT deleted automatically.

This prevents accidental data loss.

---

# Important Concepts

## Stable Pod Names

```text
nginx-0
nginx-1
nginx-2
```

Remain consistent across restarts.

---

## Stable DNS

```text
nginx-0.stateful-service.default.svc.cluster.local
nginx-1.stateful-service.default.svc.cluster.local
nginx-2.stateful-service.default.svc.cluster.local
```

Used by distributed databases for node discovery.

---

## volumeClaimTemplates

Automatically creates one PVC per pod.

Example:

```text
web-data-nginx-0
web-data-nginx-1
web-data-nginx-2
```

Provides dedicated persistent storage.

---

# Key Learnings

* StatefulSets are used for stateful applications.
* Pods receive stable names and DNS identities.
* Each replica gets its own PVC.
* Pods start in order and terminate in reverse order.
* Data survives pod deletion.
* Scaling down does not delete PVCs.
* Deleting a StatefulSet does not delete PVCs.
* Headless Services are required for StatefulSets.

---

# Conclusion

Today I learned Kubernetes StatefulSets and understood why databases require stable identities, dedicated storage, and predictable networking. StatefulSets solve problems that Deployments cannot, making them the preferred workload type for database and distributed systems.
