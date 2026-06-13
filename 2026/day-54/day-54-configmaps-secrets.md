# Day 53 – Kubernetes ConfigMaps and Secrets

## Objective

Today I learned how Kubernetes ConfigMaps and Secrets are used to provide configuration and sensitive data to applications. I practiced creating ConfigMaps and Secrets, consuming them as environment variables and volumes, updating configurations dynamically, and troubleshooting common issues.

---

# Task 1 – Create a ConfigMap Using YAML

## ConfigMap Manifest

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_PORT: "8080"
  APP_DEBUG: "false"
```

## Apply

```bash
kubectl apply -f app-config.yaml
```

## Verify

```bash
kubectl get configmap
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

---

# Task 2 – Inject ConfigMap Values Using envFrom

## Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-map-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command:
      - sh
      - -c
      - |
        echo "Environment variables from ConfigMap:"
        printenv
        sleep 3600
    envFrom:
      - configMapRef:
          name: app-config
```

## Apply

```bash
kubectl apply -f pod.yaml
```

## Verify

```bash
kubectl logs config-map-pod
```

Output:

```text
APP_DEBUG=false
APP_PORT=8080
APP_ENV=production
```

---

# Task 3 – Create a Custom Nginx Configuration

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
      listen 80;

      location /health {
        return 200 'healthy';
        add_header Content-Type text/plain;
      }
    }
```

## Nginx Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/conf.d

  volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-config
```

## Verify

```bash
kubectl port-forward pod/nginx 8080:80
curl http://localhost:8080/health
```

Expected:

```text
healthy
```

---

# Error Encountered

## curl Not Found

Command:

```bash
kubectl exec config-map-pod -- curl -s http://localhost/health
```

Error:

```text
exec: "curl": executable file not found in $PATH
```

### Cause

The container image did not contain curl.

### Solution

Use:

```bash
kubectl port-forward
```

or

```bash
kubectl exec <pod> -- wget -qO-
```

---

# Task 4 – Create a Secret

## Create Secret

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cureP@ss0rd
```

## Verify

```bash
kubectl get secret db-credentials -o yaml
```

Output:

```yaml
data:
  DB_USER: YWRtaW4=
  DB_PASSWORD: czNjdXJlUEBzczByZA==
```

---

# Understanding Base64 Encoding

Decode username:

```bash
echo YWRtaW4= | base64 -d
```

Output:

```text
admin
```

Decode password:

```bash
echo czNjdXJlUEBzczByZA== | base64 -d
```

Output:

```text
s3cureP@ss0rd
```

---

# Task 5 – Consume Secret Using Environment Variables

## Pod Example

```yaml
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: DB_USER

- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: DB_PASSWORD
```

---

# Task 6 – Mount Secret as a Volume

## Pod Manifest

```yaml
volumeMounts:
- name: db-credentials-volume
  mountPath: /etc/db-credentials
  readOnly: true

volumes:
- name: db-credentials-volume
  secret:
    secretName: db-credentials
```

## Verify

```bash
kubectl exec -it pod-name -- ls /etc/db-credentials
```

Output:

```text
DB_USER
DB_PASSWORD
```

Read values:

```bash
kubectl exec -it pod-name -- cat /etc/db-credentials/DB_USER
kubectl exec -it pod-name -- cat /etc/db-credentials/DB_PASSWORD
```

---

# MySQL Secret Troubleshooting

## Pod Status

```bash
kubectl get pods
```

Output:

```text
mysql   CrashLoopBackOff
```

## Logs

```bash
kubectl logs mysql
```

Error:

```text
Database is uninitialized and password option is not specified

MYSQL_ROOT_PASSWORD
MYSQL_ALLOW_EMPTY_PASSWORD
MYSQL_RANDOM_ROOT_PASSWORD
```

## Root Cause

Secret contained:

```yaml
DB_USER
DB_PASSWORD
```

MySQL expected:

```yaml
MYSQL_ROOT_PASSWORD
MYSQL_USER
MYSQL_PASSWORD
MYSQL_DATABASE
```

Key mismatch caused startup failure.

---

# CreateContainerConfigError

After modifying Secret references:

```text
CreateContainerConfigError
```

## Investigation

```bash
kubectl describe pod mysql
```

### Possible Causes

* Secret does not exist
* Secret name mismatch
* Secret key mismatch

---

# Pod Immutability

When adding Secret volumes to an existing Pod:

```bash
kubectl apply -f mysql.yaml
```

Error:

```text
pod updates may not change fields other than spec.containers[*].image
```

## Explanation

Pods are immutable.

You cannot modify:

* volumes
* volumeMounts
* container ports
* environment variables

after creation.

## Solution

```bash
kubectl delete pod mysql
kubectl apply -f mysql.yaml
```

Or:

```bash
kubectl replace --force -f mysql.yaml
```

---

# Task 7 – ConfigMap Update Propagation

## Create ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: live-config
data:
  message: hello
```

---

## Create Watcher Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-watcher
spec:
  containers:
  - name: busybox
    image: busybox
    command:
      - sh
      - -c
      - |
        while true; do
          cat /etc/config/message
          sleep 5
        done

    volumeMounts:
    - name: config-volume
      mountPath: /etc/config

  volumes:
  - name: config-volume
    configMap:
      name: live-config
```

---

## Update ConfigMap

```bash
kubectl patch configmap live-config \
  --type merge \
  -p '{"data":{"message":"world"}}'
```

---

## Observation

Initial:

```text
hello
```

After update:

```text
world
```

without restarting the Pod.

---

# Key Learning

## ConfigMap as Environment Variable

```yaml
envFrom:
- configMapRef:
    name: app-config
```

* Loaded only at Pod startup
* Requires Pod restart after ConfigMap update

---

## ConfigMap as Volume

```yaml
volumes:
- configMap:
    name: app-config
```

* Updates automatically
* No Pod restart required

---

# Interview Questions Learned

### Where are ConfigMaps stored?

In Kubernetes etcd.

### Where are Secrets stored?

In etcd (Base64 encoded).

### Difference Between ConfigMap and Secret?

| ConfigMap          | Secret                      |
| ------------------ | --------------------------- |
| Non-sensitive data | Sensitive data              |
| Plain text         | Base64 encoded              |
| App configuration  | Passwords, tokens, API keys |

### Does updating a ConfigMap restart Pods?

No.

### Do environment variables update automatically after ConfigMap changes?

No.

### Do volume-mounted ConfigMaps update automatically?

Yes.

### Can Pods be modified after creation?

Only a few fields like container images. Most fields are immutable.

---
