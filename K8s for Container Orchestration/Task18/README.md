# Lab 18: Control Pod-to-Pod Traffic via Network Policy

## What This Lab Does
Creates a NetworkPolicy that restricts access to the MySQL pod — **only** the Node.js app pods are allowed to connect on port 3306. All other traffic is blocked.

---

## How It Works
```
nodejs-app pods  ──── port 3306 ────▶  mysql pod  ✅ ALLOWED
any other pod    ──── port 3306 ────▶  mysql pod  ❌ BLOCKED
```

---

## File: `networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-mysql
  namespace: ivolve
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: nodejs-app
      ports:
        - protocol: TCP
          port: 3306
```

## Apply

```bash
kubectl apply -f networkpolicy.yaml
```

---

## Verify

```bash
# Check network policy exists
kubectl get networkpolicy -n ivolve

# See full details
kubectl describe networkpolicy allow-app-to-mysql -n ivolve
```

---

## Expected Result
```
NAME                 POD-SELECTOR   AGE
allow-app-to-mysql   app=mysql      1m
```
- Only pods with label `app=nodejs-app` can reach MySQL on port 3306
- All other pods are blocked from accessing MySQL
