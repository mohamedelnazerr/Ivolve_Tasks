# Lab 10: Node Isolation Using Taints in Kubernetes

## Overview

This lab demonstrates how to isolate a Kubernetes node using **Taints**. A taint is applied to one worker node with the key-value `node=worker` and effect `NoSchedule`, which prevents any pod from being scheduled onto that node unless the pod explicitly tolerates the taint.

---

## Prerequisites

- A running Kubernetes cluster with **2 nodes**
- `kubectl` configured and connected to the cluster

---

## Concepts

| Term | Description |
|------|-------------|
| **Taint** | A property applied to a node that repels pods from being scheduled on it |
| **NoSchedule** | Effect that prevents new pods from being placed on the tainted node |
| **Toleration** | A property on a pod that allows it to be scheduled on a tainted node |

---

## Steps

### Step 1 — Verify Your Cluster Has 2 Nodes

```bash
kubectl get nodes
```

Expected output:
```
NAME           STATUS   ROLES           AGE
master-node    Ready    control-plane   10m
worker-node    Ready    <none>          8m
```

> **Note:** Copy the exact name of your worker node — you will need it in the next step.

---

### Step 2 — Apply the Taint to the Worker Node

```bash
kubectl taint nodes <worker-node-name> node=worker:NoSchedule
```

Replace `<worker-node-name>` with the actual name from Step 1.

Expected output:
```
node/<worker-node-name> tainted
```

---

### Step 3 — Verify the Taint Was Applied

```bash
kubectl describe node <worker-node-name> | grep -A3 Taints
```

Expected output:
```
Taints: node=worker:NoSchedule
```

To check all nodes at once:
```bash
kubectl describe nodes | grep -A3 Taints
```

---

## Verification — Confirm NoSchedule Works

Create a simple pod without a toleration and confirm it does **not** land on the tainted node:

```bash
kubectl run test-pod --image=nginx
kubectl get pod test-pod -o wide
```

The pod should be scheduled only on the **untainted** node.

---

## How Taints and Tolerations Work

```
Without Toleration:                  With Toleration:
                                     
┌──────────────┐                     ┌──────────────┐
│  master-node │  ◀── Pod lands here │  master-node │
│  (no taint)  │                     │  (no taint)  │
└──────────────┘                     └──────────────┘

┌──────────────┐                     ┌──────────────┐
│  worker-node │  ✗ Pod blocked      │  worker-node │  ◀── Pod lands here
│  node=worker │                     │  node=worker │      (toleration added)
│  NoSchedule  │                     │  NoSchedule  │
└──────────────┘                     └──────────────┘
```

---

## Adding a Toleration to a Pod (Reference)

If you want a pod to be allowed on the tainted node, add this to the pod spec:

```yaml
spec:
  tolerations:
    - key: "node"
      operator: "Equal"
      value: "worker"
      effect: "NoSchedule"
```

---

## Useful Commands

```bash
# List all taints on all nodes
kubectl describe nodes | grep Taints

# Remove the taint (note the trailing minus sign)
kubectl taint nodes <worker-node-name> node=worker:NoSchedule-

# Check where pods are scheduled
kubectl get pods -o wide
```
