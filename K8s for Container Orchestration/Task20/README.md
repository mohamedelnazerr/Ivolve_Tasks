# Lab 20: Securing Kubernetes with RBAC and Service Accounts

## What This Lab Does
Creates a Service Account (`jenkins-sa`) with a Role that grants **read-only** access to pods only. Uses RoleBinding to attach the Role to the Service Account.

---

## How It Works
```
jenkins-sa (Service Account)
    │
    └── RoleBinding ──▶ pod-reader (Role)
                            │
                            └── can: get, list pods
                            └── cannot: delete, create, or access anything else
```

---

## File: `rbac.yaml`

```yaml
# 1. Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-sa
  namespace: ivolve
---
# 2. Secret for Service Account token
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-sa-token
  namespace: ivolve
  annotations:
    kubernetes.io/service-account.name: jenkins-sa
type: kubernetes.io/service-account-token
---
# 3. Role - read only on pods
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: ivolve
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
---
# 4. RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-sa-pod-reader
  namespace: ivolve
subjects:
  - kind: ServiceAccount
    name: jenkins-sa
    namespace: ivolve
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## Apply

```bash
kubectl apply -f rbac.yaml
```

---

## Verify

```bash
# Retrieve the service account token
kubectl get secret jenkins-sa-token -n ivolve -o jsonpath='{.data.token}' | base64 --decode

# Check jenkins-sa CAN list pods
kubectl auth can-i list pods -n ivolve --as=system:serviceaccount:ivolve:jenkins-sa

# Check jenkins-sa CANNOT delete pods
kubectl auth can-i delete pods -n ivolve --as=system:serviceaccount:ivolve:jenkins-sa

# Check jenkins-sa CANNOT access deployments
kubectl auth can-i list deployments -n ivolve --as=system:serviceaccount:ivolve:jenkins-sa
```

---

## Expected Result
```
yes   # list pods     ✅
no    # delete pods   ❌
no    # list deployments ❌
```
