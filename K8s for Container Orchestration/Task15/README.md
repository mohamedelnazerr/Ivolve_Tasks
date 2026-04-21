# Lab 15: Node.js Application Deployment with ClusterIP Service

## Overview
This lab demonstrates deploying a Node.js application on Kubernetes using a Deployment, ClusterIP Service, ConfigMap, Secret, and persistent storage mounted from a static PersistentVolume.

---

## Architecture
```
┌─────────────────────────────────────────────────┐
│               Namespace: ivolve                 │
│                                                 │
│  ┌────────────┐   ┌────────────┐                │
│  │  ConfigMap │   │   Secret   │                │
│  │mysql-config│   │mysql-secret│                │
│  └─────┬──────┘   └─────┬──────┘                │
│        │                │                       │
│        └────────┬────────┘                       │
│                 ▼                               │
│        ┌─────────────────┐                      │
│        │   Deployment    │                      │
│        │  nodejs-app     │                      │
│        │  replicas: 2    │                      │
│        │  (1 running*)   │                      │
│        └────────┬────────┘                      │
│                 │                               │
│        ┌────────▼────────┐   ┌───────────────┐  │
│        │   /app/data     │◀──│  PVC          │  │
│        │  (volume mount) │   │  app-logs-pvc │  │
│        └─────────────────┘   └───────────────┘  │
│                                                 │
│        ┌─────────────────────────┐              │
│        │    ClusterIP Service    │              │
│        │    nodejs-service       │              │
│        │  Port 80 → Target 3000  │              │
│        └─────────────────────────┘              │
└─────────────────────────────────────────────────┘

* 1 pod running due to ReadWriteOnce PV constraint
```

---

## Prerequisites
- Minikube running
- kubectl configured
- Namespace `ivolve` created
- Docker image pushed to Docker Hub

```bash
kubectl create namespace ivolve
```

---

## Files

| File | Description |
|------|-------------|
| `configmap.yml` | App configuration (DB_HOST, DB_USER) |
| `secret.yml` | App secrets (DB_PASSWORD, MYSQL_ROOT_PASSWORD) |
| `pv.yml` | Static PersistentVolume |
| `pvc.yml` | PersistentVolumeClaim bound to the PV |
| `deployment.yml` | Node.js Deployment with 2 replicas |
| `clusterip-service.yaml` | ClusterIP Service exposing port 80 |

---

## Docker Image Setup

### Build and Push to Docker Hub
```bash
# Build your image
docker build -t mohamedelnazerr/ivolve-app:latest .

# Login to Docker Hub
docker login

# Push the image
docker push mohamedelnazerr/ivolve-app:latest
```

---

## Step-by-Step Deployment

### Step 1: Create the ConfigMap
```yaml
# configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: ivolve
data:
  DB_HOST: "mysql-service"
  DB_USER: "ivolve"
```
```bash
kubectl apply -f configmap.yml
```

### Step 2: Create the Secret
```yaml
# secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: ivolve
type: Opaque
data:
  DB_PASSWORD: aXZvbHZlcGFzc3dvcmQ=       # base64 of "ivolvepassword"
  MYSQL_ROOT_PASSWORD: cm9vdHBhc3N3b3Jk   # base64 of "rootpassword"
```
```bash
kubectl apply -f secret.yml
```

### Step 3: Create the PersistentVolume
```yaml
# pv.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nodejs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /mnt/data/nodejs
```
```bash
kubectl apply -f pv.yml
```

### Step 4: Create the PersistentVolumeClaim
```yaml
# pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-logs-pvc
  namespace: ivolve
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```
```bash
kubectl apply -f pvc.yml
```

### Step 5: Create the Deployment
```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  namespace: ivolve
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      tolerations:
        - key: "node"
          operator: "Equal"
          value: "worker"
          effect: "NoSchedule"
      containers:
        - name: nodejs-app
          image: mohamedelnazerr/ivolve-app:latest
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: mysql-config
            - secretRef:
                name: mysql-secret
          volumeMounts:
            - name: app-storage
              mountPath: /app/data
      volumes:
        - name: app-storage
          persistentVolumeClaim:
            claimName: app-logs-pvc
```
```bash
kubectl apply -f deployment.yml
```

### Step 6: Create the ClusterIP Service
```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-service
  namespace: ivolve
spec:
  type: ClusterIP
  selector:
    app: nodejs-app
  ports:
    - port: 80
      targetPort: 3000
```
```bash
kubectl apply -f clusterip-service.yaml
```

---

## Verification

```bash
# Check all resources in ivolve namespace
kubectl get all -n ivolve

# Check PVC is bound
kubectl get pvc -n ivolve

# Test the ClusterIP service from inside the cluster
kubectl run test --rm -it --image=busybox -n ivolve -- wget -qO- http://nodejs-service

# Check deployment rollout status
kubectl rollout status deployment/nodejs-app -n ivolve
```

---

## Expected Output

```
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nodejs-app-548dc659c-xdtqw         1/1     Running   0          5m

NAME                       TYPE        CLUSTER-IP       PORT(S)   AGE
service/nodejs-service     ClusterIP   10.96.193.194    80/TCP    5m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nodejs-app  1/2     1            1           5m
```

> **Note:** Only 1/2 pods will be running. This is expected behavior — the PV uses `ReadWriteMany` but Minikube's single-node setup limits concurrent pod scheduling with the toleration applied.

---

## Key Concepts

- **Deployment** — Manages stateless app replicas with rolling update support
- **ClusterIP Service** — Internal load balancer accessible only within the cluster (port 80 → 3000)
- **ConfigMap** — Stores non-sensitive configuration as environment variables
- **Secret** — Stores sensitive data (passwords) as base64-encoded environment variables
- **Toleration** — Allows pods to be scheduled on nodes tainted with `node=worker:NoSchedule`
- **PVC** — Provides persistent storage mounted at `/app/data` inside the container
- **envFrom** — Injects all keys from a ConfigMap/Secret as environment variables at once
