# Lab 11: Namespace Management and Resource Quota Enforcement

## Overview

This lab covers Kubernetes **Namespace** creation and **ResourceQuota** enforcement. A namespace called `ivolve` is created and a quota is applied that limits the total number of running pods inside it to **2 pods** maximum.

---

## Prerequisites

- A running Kubernetes cluster
- `kubectl` configured and connected to the cluster

---

## Concepts

| Term | Description |
|------|-------------|
| **Namespace** | A virtual cluster inside Kubernetes used to isolate resources |
| **ResourceQuota** | A policy that limits total resource consumption within a namespace |
| **pods (hard limit)** | The maximum number of pods allowed to exist in the namespace |

---

## Manifest File

Create a file named `lab11-namespace-quota.yml` with the following content:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ivolve
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-quota
  namespace: ivolve
spec:
  hard:
    pods: "2"
```

---

## Steps

### Step 1 — Apply the Manifest

```bash
kubectl apply -f lab11-namespace-quota.yml
```

Expected output:
```
namespace/ivolve created
resourcequota/pod-quota created
```

---

### Step 2 — Verify the Namespace

```bash
kubectl get namespace ivolve
```

Expected output:
```
NAME     STATUS   AGE
ivolve   Active   5s
```

---

### Step 3 — Verify the ResourceQuota

```bash
kubectl get resourcequota -n ivolve
```

```bash
kubectl describe resourcequota pod-quota -n ivolve
```

Expected output:
```
Name:       pod-quota
Namespace:  ivolve
Resource    Used  Hard
--------    ----  ----
pods        0     2
```

---

### Step 4 — Test the Quota Enforcement

Create two pods — both should succeed:

```bash
kubectl run pod1 --image=nginx -n ivolve
kubectl run pod2 --image=nginx -n ivolve
```

Try to create a third pod — this should **fail**:

```bash
kubectl run pod3 --image=nginx -n ivolve
```

Expected error:
```
Error from server (Forbidden): pods "pod3" is forbidden:
exceeded quota: pod-quota, requested: pods=1,
used: pods=2, limited: pods=2
```

Clean up test pods:
```bash
kubectl delete pod pod1 pod2 -n ivolve
```

---

## How Resource Quota Enforcement Works

```
Namespace: ivolve
┌────────────────────────────────────────┐
│                                        │
│  ┌─────────┐  ┌─────────┐             │
│  │  pod1   │  │  pod2   │  ✓ Allowed  │
│  │ (nginx) │  │ (nginx) │             │
│  └─────────┘  └─────────┘             │
│                                        │
│  ┌─────────┐                           │
│  │  pod3   │  ✗ Blocked — quota full  │
│  │ (nginx) │                           │
│  └─────────┘                           │
│                                        │
│  ResourceQuota: pods hard limit = 2    │
└────────────────────────────────────────┘
```

---

## Useful Commands

```bash
# List all namespaces
kubectl get namespaces

# List all pods in the ivolve namespace
kubectl get pods -n ivolve

# Check current quota usage
kubectl describe resourcequota pod-quota -n ivolve

# Delete the namespace (removes everything inside it)
kubectl delete namespace ivolve
```
