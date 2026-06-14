# Day 55 – Persistent Volumes (PV) and Persistent Volume Claims (PVC)

## Why Containers Need Persistent Storage

Containers are ephemeral. When a Pod is deleted, restarted, or rescheduled, its filesystem and any data written to it disappear. For databases, file uploads, or any stateful workload, this is unacceptable. Kubernetes solves this by decoupling storage from the Pod lifecycle using **PersistentVolumes (PV)** and **PersistentVolumeClaims (PVC)**.

---

## Task 1: See the Problem — Data Lost on Pod Deletion

### Manifest (`emptydir-pod.yaml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "date > /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: data-vol
      mountPath: /data
  volumes:
  - name: data-vol
    emptyDir: {}
```

### Commands

```bash
kubectl apply -f emptydir-pod.yaml
kubectl exec emptydir-demo -- cat /data/message.txt
# Sun Jun 14 10:23:01 UTC 2026

kubectl delete pod emptydir-demo
kubectl apply -f emptydir-pod.yaml
kubectl exec emptydir-demo -- cat /data/message.txt
# Sun Jun 14 10:25:47 UTC 2026
```

### Verify: Is the timestamp the same or different after recreation?

**Different.** `emptyDir` volumes are tied to the Pod's lifecycle. When the Pod is deleted, the volume and its contents are permanently destroyed. The new Pod starts with a fresh, empty volume and writes a new timestamp.

---

## Task 2: Create a PersistentVolume (Static Provisioning)

### Manifest (`pv.yml`)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/k8s-pv-data
```

### Commands

```bash
mkdir -p /tmp/k8s-pv-data
kubectl apply -f pv.yml
kubectl get pv
```

```
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
pv-demo   1Gi        RWO            Retain           Available           manual         5s
```

### Access Modes Reference

| Mode | Meaning |
|---|---|
| `ReadWriteOnce (RWO)` | Read-write by a single node |
| `ReadOnlyMany (ROX)` | Read-only by many nodes |
| `ReadWriteMany (RWX)` | Read-write by many nodes |

### Verify: What is the STATUS of the PV?

**`Available`** — the PV is created and ready to be claimed, but no PVC has bound to it yet.

---

## Task 3: Create a PersistentVolumeClaim

### Manifest (`pcv.yml`)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
```

### Commands

```bash
kubectl apply -f pcv.yml
kubectl get pvc
```

```
NAME       STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-demo   Bound    pv-demo    1Gi        RWO            manual         3s
```

```bash
kubectl get pv
```

```
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   AGE
pv-demo   1Gi        RWO            Retain           Bound    default/pvc-demo   manual         2m
```

### Troubleshooting Encountered

- PVC stayed `Pending` with `WaitForFirstConsumer` when no `storageClassName` was set (it picked up the cluster default `standard` StorageClass instead of matching `pv-demo`).
- PVC stayed `Pending` with `storageclass.storage.k8s.io "manual" not found` when `storageClassName: manual` was set but the PV hadn't been (re)applied with a matching `storageClassName`, or apply order was PVC-before-PV.
- **Fix**: Ensure both PV and PVC use the same `storageClassName: manual`, and apply the PV first so it's `Available` before the PVC is created.

### Verify: What does the VOLUME column in `kubectl get pvc` show?

**`pv-demo`** — the name of the PersistentVolume the PVC got bound to. Although the PVC requested only 500Mi, Kubernetes matched it to the smallest *available* PV satisfying the capacity and access mode requirements (1Gi `pv-demo`), so the CAPACITY column reflects the bound PV's actual size, not the requested amount.

---

## Task 4: Use the PVC in a Pod — Data That Survives

### Manifest (`pvc-pod.yaml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo \"Pod started at $(date)\" >> /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: data-vol
      mountPath: /data
  volumes:
  - name: data-vol
    persistentVolumeClaim:
      claimName: pvc-demo
```

### Commands

```bash
kubectl apply -f pvc-pod.yaml
kubectl exec pvc-demo-pod -- cat /data/message.txt
```

```
Pod started at Sun Jun 14 11:02:10 UTC 2026
```

```bash
kubectl delete pod pvc-demo-pod
kubectl apply -f pvc-pod.yaml
kubectl exec pvc-demo-pod -- cat /data/message.txt
```

```
Pod started at Sun Jun 14 11:02:10 UTC 2026
Pod started at Sun Jun 14 11:05:43 UTC 2026
```

### Verify: Does the file contain data from both the first and second Pod?

**Yes.** Deleting the Pod only removes the Pod object — the PVC and its bound PV (backed by `/tmp/k8s-pv-data` with `Retain` policy) persist independently. The recreated Pod mounts the same PV and appends to the existing file, so both timestamps are present.

---

## Task 5: StorageClasses and Dynamic Provisioning

### Commands

```bash
kubectl get storageclass
kubectl describe storageclass standard
```

```
NAME                 PROVISIONER               RECLAIMPOLICY   VOLUMEBINDINGMODE      AGE
standard (default)   k8s.io/minikube-hostpath  Delete          WaitForFirstConsumer   10d
```

### What This Means

- **Provisioner**: `k8s.io/minikube-hostpath` — automatically creates the underlying storage when a PVC requests this class.
- **Reclaim Policy**: `Delete` — when the PVC is deleted, the auto-created PV and its storage are deleted too.
- **Volume Binding Mode**: `WaitForFirstConsumer` — the PVC stays `Pending` until a Pod that uses it is scheduled, so the volume can be provisioned with the correct topology.

With dynamic provisioning, developers only need to create a PVC — the StorageClass handles PV creation automatically, no admin step required.

### Verify: What is the default StorageClass in your cluster?

**`standard`** (provisioner `k8s.io/minikube-hostpath`).

---

## Task 6: Dynamic Provisioning

### Manifest (`pvc-dynamic.yaml`)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi
```

### Commands

```bash
kubectl apply -f pvc-dynamic.yaml
kubectl describe pvc pvc-dynamic
```

```
Status:  Pending
Events:
  Normal  WaitForFirstConsumer  waiting for first consumer to be created before binding
```

This is **expected**, not an error — `standard` uses `WaitForFirstConsumer` binding mode, so the PVC waits for a Pod.

### Pod Manifest (`pvc-dynamic-pod.yaml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-dynamic-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo \"Dynamic PV data at $(date)\" >> /data/message.txt && sleep 3600"]
    volumeMounts:
    - name: data-vol
      mountPath: /data
  volumes:
  - name: data-vol
    persistentVolumeClaim:
      claimName: pvc-dynamic
```

### Commands

```bash
kubectl apply -f pvc-dynamic-pod.yaml
kubectl get pvc
```

```
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-demo      Bound    pv-demo                                    1Gi        RWO            manual         20m
pvc-dynamic   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO            standard       5s
```

```bash
kubectl get pv
```

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   AGE
pv-demo                                    1Gi        RWO            Retain           Bound    default/pvc-demo       manual         20m
pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO            Delete           Bound    default/pvc-dynamic    standard       5s
```

```bash
kubectl exec pvc-dynamic-pod -- cat /data/message.txt
```

```
Dynamic PV data at Sun Jun 14 11:22:30 UTC 2026
```

### Verify: How many PVs exist now? Which was manual, which was dynamic?

**Two PVs exist:**

| PV Name | Type | StorageClass | Reclaim Policy | Created |
|---|---|---|---|---|
| `pv-demo` | Manual (static) | `manual` | `Retain` | Pre-created by admin in Task 2 |
| `pvc-xxxxxxxx-...` (auto-named, matches PVC UID) | Dynamic | `standard` | `Delete` | Auto-provisioned when `pvc-dynamic-pod` was scheduled |

---

## Task 7: Clean Up

### Commands

```bash
# 1. Delete all pods first
kubectl delete pod emptydir-demo pvc-demo-pod pvc-dynamic-pod

# 2. Delete PVCs
kubectl delete pvc pvc-demo pvc-dynamic

# Check PVs
kubectl get pv
```

```
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM              STORAGECLASS   AGE
pv-demo   1Gi        RWO            Retain           Released   default/pvc-demo   manual         40m
```

```bash
# 4. Delete the remaining PV
kubectl delete pv pv-demo
```

### Verify: Which PV was auto-deleted and which was retained? Why?

- The **dynamic PV** (`standard` StorageClass, `Delete` reclaim policy) was **automatically deleted** the moment its PVC (`pvc-dynamic`) was deleted. The `Delete` policy tells the provisioner to remove the underlying storage entirely — appropriate for ephemeral/on-demand storage.

- The **manual PV** (`pv-demo`, `Retain` reclaim policy) became **`Released`** after `pvc-demo` was deleted. The PV and its data on `/tmp/k8s-pv-data` were preserved — Kubernetes does not automatically delete it or its contents. It required an explicit `kubectl delete pv pv-demo` to remove. `Retain` exists specifically to prevent accidental data loss, even when the claim referencing it is gone.

---

## Summary / Key Takeaways

- **`emptyDir`**: ephemeral, tied to the Pod's lifecycle — data is lost on Pod deletion.
- **PV**: cluster-wide storage resource, not namespaced. Lifecycle: `Available` → `Bound` → `Released` → deleted/reclaimed.
- **PVC**: namespaced request for storage, matched to a PV by capacity, access mode, and storage class.
- **Static provisioning**: admin pre-creates PVs manually.
- **Dynamic provisioning**: StorageClass's provisioner auto-creates PVs when a PVC requests it — developers only manage PVCs.
- **`WaitForFirstConsumer`**: delays binding/provisioning until a Pod actually consumes the PVC, ensuring correct topology.
- **Access modes**: `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany` — control how many nodes can mount the volume and in what mode.
- **Reclaim policies**: `Retain` (preserve data, manual cleanup) vs `Delete` (auto-remove storage with the PVC).
