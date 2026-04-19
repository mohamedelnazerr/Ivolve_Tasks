# Lab 13: Persistent Storage Setup for Application Logging

## Overview

This lab sets up **persistent storage** in Kubernetes so that application logs survive pod restarts and deletions. A **PersistentVolume (PV)** is created using `hostPath` storage on the node, and a **PersistentVolumeClaim (PVC)** is created to request and bind to that storage.

---

## Prerequisites

- Minikube running (`minikube start`)
- `kubectl` configured and connected to the cluster

---

## Concepts

| Term | Description |
|------|-------------|
| **PersistentVolume (PV)** | A piece of storage provisioned in the cluster by an admin. It exists independently of any pod. |
| **PersistentVolumeClaim (PVC)** | A request for storage made by a pod. It binds to a matching PV. |
| **hostPath** | A storage type that mounts a directory from the node's filesystem into the pod. |
| **ReadWriteMany** | Access mode that allows multiple pods/nodes to read and write simultaneously. |
| **Retain** | Reclaim policy that keeps the PV data even after the PVC is deleted. |

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Kubernetes Node                                │
│                                                 │
│  /mnt/app-logs  (permission 777)                │
│        │                                        │
│        ▼                                        │
│  ┌─────────────────┐       ┌─────────────────┐ │
│  │  PersistentVolume│◀─────│  PersistentVolume│ │
│  │  app-logs-pv    │ bound │  Claim           │ │
│  │  1Gi / hostPath │       │  app-logs-pvc    │ │
│  │  ReadWriteMany  │       │  1Gi             │ │
│  │  Retain         │       │  ReadWriteMany   │ │
│  └─────────────────┘       └────────┬─────────┘ │
│                                     │           │
│                              ┌──────▼──────┐   │
│                              │   App Pod   │   │
│                              │  (mounts    │   │
│                              │   the PVC)  │   │
│                              └─────────────┘   │
└─────────────────────────────────────────────────┘
```

---

## Step 1 — Create the /mnt/app-logs Directory on the Node

Since we are using `hostPath`, the directory must exist on the node itself with the correct permissions.

**If using Minikube**, SSH into the node first:

```bash
minikube ssh
```

Then inside the node run:

```bash
sudo mkdir -p /mnt/app-logs
sudo chmod 777 /mnt/app-logs
exit
```

**If using a real cluster**, run the same commands directly on the worker node via SSH:

```bash
ssh user@<worker-node-ip>
sudo mkdir -p /mnt/app-logs
sudo chmod 777 /mnt/app-logs
exit
```

---

## Step 2 — Create the PersistentVolume

Create a file named `lab13-pv.yml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-logs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/app-logs
```

Apply it:

```bash
kubectl apply -f lab13-pv.yml
```

Expected output:
```
persistentvolume/app-logs-pv created
```

---

## Step 3 — Create the PersistentVolumeClaim

Create a file named `lab13-pvc.yml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-logs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Apply it:

```bash
kubectl apply -f lab13-pvc.yml
```

Expected output:
```
persistentvolumeclaim/app-logs-pvc created
```

---

## Step 4 — Verify the PV and PVC Are Bound

```bash
kubectl get pv
kubectl get pvc
```

Expected output:
```
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
app-logs-pv    1Gi        RWX            Retain           Bound    default/app-logs-pvc

NAME            STATUS   VOLUME         CAPACITY   ACCESS MODES
app-logs-pvc    Bound    app-logs-pv    1Gi        RWX
```

> **Important:** The STATUS must say `Bound` on both. If the PVC shows `Pending`, see the Troubleshooting section below.

---

## Step 5 — Mount the PVC into a Pod (Reference)

To actually use the storage, mount the PVC inside a pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: logs-storage
          mountPath: /app/logs        # path inside the container
  volumes:
    - name: logs-storage
      persistentVolumeClaim:
        claimName: app-logs-pvc       # must match the PVC name
```

Apply it:

```bash
kubectl apply -f pod.yml
```

Verify logs are being written to the persistent path:

```bash
# Check inside the pod
kubectl exec -it app-pod -- ls /app/logs

# Check on the node directly (minikube)
minikube ssh
ls /mnt/app-logs
```

---

## Troubleshooting

**PVC stuck in `Pending` status:**
```bash
kubectl describe pvc app-logs-pvc
```
Most common causes:
- The access mode on the PVC does not match the PV (`ReadWriteMany` on both is required)
- The storage size requested by the PVC exceeds what the PV offers
- The `/mnt/app-logs` directory does not exist on the node

**Check PV details:**
```bash
kubectl describe pv app-logs-pv
```

---

## Useful Commands

```bash
# List all PVs in the cluster
kubectl get pv

# List all PVCs in the current namespace
kubectl get pvc

# Get detailed info on the PV
kubectl describe pv app-logs-pv

# Get detailed info on the PVC
kubectl describe pvc app-logs-pvc

# Delete the PVC (PV will stay because of Retain policy)
kubectl delete pvc app-logs-pvc

# Delete the PV
kubectl delete pv app-logs-pv
```

---

## Key Differences — PV vs PVC

| | PersistentVolume (PV) | PersistentVolumeClaim (PVC) |
|---|---|---|
| **Created by** | Cluster admin | Developer / app |
| **What it is** | The actual storage resource | A request for storage |
| **Scope** | Cluster-wide (no namespace) | Namespaced |
| **Lifecycle** | Independent of pods | Bound to a PV when matched |

---

## Reclaim Policy — What Happens When PVC is Deleted

| Policy | Behavior |
|--------|----------|
| `Retain` | PV and data are **kept**. Must be manually cleaned up. Used in this lab. |
| `Delete` | PV and underlying storage are **deleted automatically**. |
| `Recycle` | Data is scrubbed and PV becomes available again. Deprecated. |
