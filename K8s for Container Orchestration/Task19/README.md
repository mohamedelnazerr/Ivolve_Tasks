# Lab 19: Node-Wide Pod Management with DaemonSet

## What This Lab Does
Creates a DaemonSet in the `monitoring` namespace that deploys a **Prometheus node-exporter** pod on **every node** in the cluster. It tolerates all taints so no node is skipped.

---

## How It Works
```
Every Node in Cluster:
  ┌─────────────────────────┐
  │  node-exporter pod      │
  │  port: 9100             │
  │  /metrics endpoint      │
  └─────────────────────────┘
```

---

## Step 1: Create the Namespace

```bash
kubectl create namespace monitoring
```

---

## File: `daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      tolerations:
        - operator: "Exists"    # tolerates ALL existing taints
      hostNetwork: true
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          ports:
            - containerPort: 9100
              hostPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
```

## Apply

```bash
kubectl apply -f daemonset.yaml
```

---

## Verify

```bash
# Check pod is running on each node
kubectl get pods -n monitoring -o wide

# Access metrics endpoint
minikube ssh
curl http://localhost:9100/metrics | head -20
```

---

## Expected Result
```
NAME                    READY   STATUS    NODE
node-exporter-xxxxx     1/1     Running   minikube
```
- One pod per node
- Metrics accessible at `http://<node-ip>:9100/metrics`
