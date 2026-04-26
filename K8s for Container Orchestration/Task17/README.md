# Lab 17: Pod Resource Management with CPU and Memory Requests and Limits

## What This Lab Does
Updates the existing Node.js Deployment to set CPU and memory **requests** (minimum guaranteed) and **limits** (maximum allowed) for the container.

---

## Resource Values

| Type | CPU | Memory |
|------|-----|--------|
| Request | 1 vCPU | 1Gi |
| Limit | 2 vCPUs | 2Gi |

---

## Changes Made
Added `resources` block inside the container spec in `deployment.yml`:

```yaml
containers:
  - name: nodejs-app
    image: mohamedelnazerr/ivolve-app:latest
    ports:
      - containerPort: 3000
    resources:
      requests:
        cpu: "1"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "2Gi"
```

## Apply

```bash
kubectl apply -f deployment.yml
```

---

## Verify

```bash
# Check requests and limits are applied
kubectl describe pod -n ivolve -l app=nodejs-app | grep -A8 "Limits\|Requests"

# Monitor real-time resource usage (requires metrics-server)
minikube addons enable metrics-server
kubectl top pod -n ivolve
```

---

## Expected Result
```
Limits:
  cpu:     2
  memory:  2Gi
Requests:
  cpu:     1
  memory:  1Gi
```
