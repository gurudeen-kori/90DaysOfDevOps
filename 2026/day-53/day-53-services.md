# Day 53 – Kubernetes Services

## Objective

Today I learned how Kubernetes Services provide stable networking for Pods. Since Pod IPs change whenever Pods restart, Services provide a stable IP address, DNS name, and load balancing mechanism for applications running inside Deployments.

---

# Why Services?

Every Pod gets its own IP address.

Problems:

* Pod IPs are not stable and change when Pods restart.
* A Deployment runs multiple Pods, making it difficult to know which Pod IP to connect to.

A Service solves these problems by providing:

* Stable IP Address
* Stable DNS Name
* Load Balancing Across Pods

```text
Client
   │
   ▼
Service (Stable IP)
   │
   ├── Pod 1
   ├── Pod 2
   └── Pod 3
```

---

# Task 1: Deploy the Application

## Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: web-app
  labels:
    app: web-app

spec:
  replicas: 3

  selector:
    matchLabels:
      app: web-app

  template:
    metadata:
      labels:
        app: web-app

    spec:
      containers:
      - name: nginx
        image: nginx:1.25

        ports:
        - containerPort: 80
```

Apply the Deployment:

```bash
kubectl apply -f app-deployment.yaml
```

Verify Pods:

```bash
kubectl get pods -o wide
```

Example Output:

```text
NAME                        READY   STATUS    IP
web-app-abcde               1/1     Running   10.244.1.2
web-app-fghij               1/1     Running   10.244.2.2
web-app-klmno               1/1     Running   10.244.2.3
```

---

# Task 2: ClusterIP Service

ClusterIP is the default Service type.

It allows communication only inside the Kubernetes cluster.

## ClusterIP Manifest

```yaml
apiVersion: v1
kind: Service

metadata:
  name: webapp-service

spec:
  type: ClusterIP

  selector:
    app: web-app

  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f clusterip-service.yaml
```

Verify:

```bash
kubectl get svc
```

Example Output:

```text
NAME             TYPE        CLUSTER-IP
webapp-service   ClusterIP   10.96.129.246
```

---

## Testing ClusterIP Service

Create a temporary test pod:

```bash
kubectl run test-client \
--image=busybox:latest \
-it --rm \
--restart=Never -- sh
```

Inside the pod:

```bash
wget -qO- http://webapp-service
```

Output:

```html
Welcome to nginx!
```

Result:

* Service reachable
* Traffic successfully forwarded to backend Pods

---

# Task 3: Service Discovery Using DNS

Kubernetes automatically creates DNS records for Services.

DNS Format:

```text
<service-name>.<namespace>.svc.cluster.local
```

Example:

```text
webapp-service.day-53.svc.cluster.local
```

Test DNS:

```bash
nslookup webapp-service
```

Example Output:

```text
Name: webapp-service.day-53.svc.cluster.local
Address: 10.96.129.246
```

Observation:

* DNS resolved successfully
* Returned IP matches the Service ClusterIP

---

# Task 4: NodePort Service

NodePort exposes an application outside the cluster.

Traffic Flow:

```text
Browser
   │
   ▼
NodeIP:30080
   │
   ▼
NodePort Service
   │
   ▼
Pods
```

## NodePort Manifest

```yaml
apiVersion: v1
kind: Service

metadata:
  name: web-app-nodeport

spec:
  type: NodePort

  selector:
    app: web-app

  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

Apply:

```bash
kubectl apply -f nodeport-service.yaml
```

Verify:

```bash
kubectl get svc
```

Example:

```text
NAME               TYPE       PORT(S)
web-app-nodeport   NodePort   80:30080/TCP
```

Access:

```bash
curl http://<NodeIP>:30080
```

Expected Output:

```html
Welcome to nginx!
```

---

# Task 5: LoadBalancer Service

LoadBalancer is used in cloud environments.

Examples:

* AWS Elastic Load Balancer
* Azure Load Balancer
* Google Cloud Load Balancer

## LoadBalancer Manifest

```yaml
apiVersion: v1
kind: Service

metadata:
  name: web-app-loadbalancer

spec:
  type: LoadBalancer

  selector:
    app: web-app

  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f loadbalancer-service.yaml
```

Verify:

```bash
kubectl get svc
```

Example:

```text
NAME                    TYPE           EXTERNAL-IP
web-app-loadbalancer    LoadBalancer   <pending>
```

### Why is EXTERNAL-IP Pending?

Local clusters such as:

* Kind
* Minikube
* Docker Desktop

do not have a cloud provider.

Therefore Kubernetes cannot provision a real external load balancer.

Result:

```text
EXTERNAL-IP = <pending>
```

---

# Understanding Service Types

| Service Type | Accessible From      | Use Case                       |
| ------------ | -------------------- | ------------------------------ |
| ClusterIP    | Inside Cluster       | Internal Service Communication |
| NodePort     | NodeIP:NodePort      | Development and Testing        |
| LoadBalancer | Public Load Balancer | Production Applications        |

---

# Service Hierarchy

```text
LoadBalancer
     │
     ▼
NodePort
     │
     ▼
ClusterIP
```

A LoadBalancer Service automatically includes:

* ClusterIP
* NodePort
* External Load Balancer

---

# Endpoints

Endpoints represent the actual Pod IP addresses behind a Service.

Check Endpoints:

```bash
kubectl get endpoints webapp-service
```

Example:

```text
NAME             ENDPOINTS
webapp-service   10.244.1.2:80,10.244.2.2:80,10.244.2.3:80
```

Architecture:

```text
webapp-service
      │
      ├── 10.244.1.2:80
      ├── 10.244.2.2:80
      └── 10.244.2.3:80
```

The Service distributes traffic across all healthy Pods.

---

# Important Service Fields

## Selector

Matches Pods using labels.

```yaml
selector:
  app: web-app
```

If labels do not match:

```text
Endpoints: <none>
```

and the Service cannot forward traffic.

---

## Port vs TargetPort

```yaml
ports:
- port: 80
  targetPort: 80
```

### port

The port exposed by the Service.

### targetPort

The port on the Pod receiving traffic.

Traffic Flow:

```text
Service:80
     │
     ▼
Pod:80
```

---

# Commands Used

### Create Resources

```bash
kubectl apply -f app-deployment.yaml
kubectl apply -f clusterip-service.yaml
kubectl apply -f nodeport-service.yaml
kubectl apply -f loadbalancer-service.yaml
```

### Verify Resources

```bash
kubectl get pods -o wide
kubectl get svc
kubectl get endpoints
kubectl describe svc webapp-service
```

### Test Connectivity

```bash
kubectl run test-client \
--image=busybox \
-it --rm \
--restart=Never -- sh

wget -qO- http://webapp-service

nslookup webapp-service
```

---

# Key Learnings

* Pods are ephemeral and their IP addresses change.
* Services provide stable networking.
* ClusterIP is used for internal communication.
* NodePort exposes applications through Node IP and Port.
* LoadBalancer is used for public access in cloud environments.
* Kubernetes DNS automatically resolves Service names.
* Endpoints represent the actual Pods behind a Service.
* Services provide built-in load balancing.

---

# Conclusion

Today I learned how Kubernetes Services provide stable communication between applications and Pods. I created and tested ClusterIP, NodePort, and LoadBalancer Services, verified DNS-based Service discovery, inspected Endpoints, and understood how Services load balance traffic across multiple Pods.
