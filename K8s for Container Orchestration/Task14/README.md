# Lab 14: StatefulSet with Headless Service (MySQL)

## Overview
This lab demonstrates deploying a MySQL database using a Kubernetes StatefulSet with a Headless Service, persistent storage, and secret-based credential management.

---

## Architecture
```
┌─────────────────────────────────────────┐
│            Namespace: ivolve            │
│                                         │
│  ┌──────────┐      ┌─────────────────┐  │
│  │  Secret  │─────▶│  StatefulSet    │  │
│  │  mysql-  │      │  mysql-0 (1/1)  │  │
│  │  secret  │      └────────┬────────┘  │
│  └──────────┘               │           │
│                             │           │
│  ┌──────────┐      ┌────────▼────────┐  │
│  │   PVC    │─────▶│  /var/lib/mysql │  │
│  │app-logs- │      └─────────────────┘  │
│  │   pvc    │                           │
│  └──────────┘      ┌─────────────────┐  │
│                    │ Headless Service │  │
│                    │  ClusterIP:None  │  │
│                    │   Port: 3306     │  │
│                    └─────────────────┘  │
└─────────────────────────────────────────┘
```

---

## Prerequisites
- Minikube running
- kubectl configured
- Namespace `ivolve` created

```bash
kubectl create namespace ivolve
```

---

## Files

| File | Description |
|------|-------------|
| `secret.yml` | MySQL root password stored as a K8s Secret |
| `pvc.yml` | PersistentVolumeClaim for MySQL data storage |
| `statefulset.yaml` | MySQL StatefulSet with 1 replica |
| `headless-service.yaml` | Headless Service (ClusterIP: None) |

---

## Step-by-Step Deployment

### Step 1: Create the Secret
```yaml
# secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: ivolve
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: cm9vdHBhc3N3b3Jk   # base64 of "rootpassword"
  DB_PASSWORD: aXZvbHZlcGFzc3dvcmQ=       # base64 of "ivolvepassword"
```
```bash
kubectl apply -f secret.yml
```

### Step 2: Create the PersistentVolumeClaim
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

### Step 3: Create the StatefulSet
```yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: ivolve
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: "mysql-headless"
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      tolerations:
        - key: "node"
          operator: "Equal"
          value: "worker"
          effect: "NoSchedule"
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: app-logs-pvc
```
```bash
kubectl apply -f statefulset.yaml
```

### Step 4: Create the Headless Service
```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: ivolve
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```
```bash
kubectl apply -f headless-service.yaml
```

---

## Verification

```bash
# Check all resources in ivolve namespace
kubectl get all -n ivolve

# Verify PVC is bound
kubectl get pvc -n ivolve

# Connect to MySQL to confirm it's operational
kubectl exec -it mysql-0 -n ivolve -- mysql -u root -p
# Enter password: rootpassword

# Inside MySQL shell
show databases;
exit
```

---

## Expected Output

```
NAME          READY   STATUS    RESTARTS   AGE
pod/mysql-0   1/1     Running   0          5m

NAME                       TYPE        CLUSTER-IP   PORT(S)    AGE
service/mysql-headless     ClusterIP   None         3306/TCP   5m
```

---

## Key Concepts

- **StatefulSet** — Manages stateful applications, provides stable pod names (mysql-0, mysql-1...)
- **Headless Service** — `clusterIP: None` allows direct DNS-based pod addressing
- **Toleration** — Allows pod to be scheduled on nodes tainted with `node=worker:NoSchedule`
- **PVC** — Provides persistent storage that survives pod restarts
- **Secret** — Securely stores sensitive credentials (base64 encoded)
