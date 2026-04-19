# Lab 12: Managing Configuration and Sensitive Data with ConfigMaps and Secrets

## Overview

This lab demonstrates how to manage application configuration in Kubernetes using **ConfigMaps** for non-sensitive data and **Secrets** for sensitive credentials. Both objects are created in the `ivolve` namespace and are designed to be injected into the Node.js application as environment variables.

---

## Prerequisites

- A running Kubernetes cluster
- `kubectl` configured and connected to the cluster
- The `ivolve` namespace already created (see Lab 11)

---

## Concepts

| Object | Purpose | Encoding |
|--------|---------|----------|
| **ConfigMap** | Stores non-sensitive configuration (hostnames, usernames) | Plain text |
| **Secret** | Stores sensitive credentials (passwords, tokens) | base64 encoded |

---

## Step 1 — Generate base64 Values for the Secret

Kubernetes Secrets require values to be base64-encoded. Run these commands to generate the encoded values:

```bash
# Encode DB_PASSWORD
echo -n "ivolvepassword" | base64
# Output: aXZvbHZlcGFzc3dvcmQ=

# Encode MYSQL_ROOT_PASSWORD
echo -n "rootpassword" | base64
# Output: cm9vdHBhc3N3b3Jk
```

> **Important:** Always use `echo -n` (the `-n` flag suppresses the trailing newline). Without it, the newline character gets encoded into the value and the password will not match at runtime.

---

## Step 2 — Create the ConfigMap

Create a file named `lab12-configmap.yml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: ivolve
data:
  DB_HOST: "mysql-service"   # hostname of the MySQL StatefulSet service
  DB_USER: "ivolve"          # user that connects to the ivolve database
```

Apply it:

```bash
kubectl apply -f lab12-configmap.yml
```

Expected output:
```
configmap/mysql-config created
```

---

## Step 3 — Create the Secret

Create a file named `lab12-secret.yml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: ivolve
type: Opaque
data:
  DB_PASSWORD: aXZvbHZlcGFzc3dvcmQ=        # base64 of "ivolvepassword"
  MYSQL_ROOT_PASSWORD: cm9vdHBhc3N3b3Jk    # base64 of "rootpassword"
```

Apply it:

```bash
kubectl apply -f lab12-secret.yml
```

Expected output:
```
secret/mysql-secret created
```

---

## Step 4 — Verify Both Objects

**Verify the ConfigMap:**
```bash
kubectl get configmap mysql-config -n ivolve -o yaml
```

**Verify the Secret:**
```bash
kubectl get secret mysql-secret -n ivolve -o yaml
```

**Decode and confirm the secret values are correct:**
```bash
# Decode DB_PASSWORD
kubectl get secret mysql-secret -n ivolve \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# Expected output: ivolvepassword

# Decode MYSQL_ROOT_PASSWORD
kubectl get secret mysql-secret -n ivolve \
  -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d
# Expected output: rootpassword
```

---

## Step 5 — Inject into the Application Pod

Use `envFrom` in your pod or deployment spec to automatically load all keys from both the ConfigMap and the Secret as environment variables:

```yaml
spec:
  containers:
    - name: app
      image: <YOUR_DOCKERHUB_USERNAME>/ivolve-app:latest
      envFrom:
        - configMapRef:
            name: mysql-config
        - secretRef:
            name: mysql-secret
```

This injects the following environment variables into the container:

| Variable | Source | Value |
|----------|--------|-------|
| `DB_HOST` | ConfigMap | `mysql-service` |
| `DB_USER` | ConfigMap | `ivolve` |
| `DB_PASSWORD` | Secret | `ivolvepassword` |
| `MYSQL_ROOT_PASSWORD` | Secret | `rootpassword` |

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│  Namespace: ivolve                               │
│                                                  │
│  ┌─────────────────────┐  ┌──────────────────┐  │
│  │  ConfigMap          │  │  Secret          │  │
│  │  mysql-config       │  │  mysql-secret    │  │
│  │─────────────────────│  │──────────────────│  │
│  │  DB_HOST            │  │  DB_PASSWORD     │  │
│  │  DB_USER            │  │  MYSQL_ROOT_     │  │
│  │  (plain text)       │  │  PASSWORD        │  │
│  └──────────┬──────────┘  │  (base64)        │  │
│             │             └────────┬─────────┘  │
│             └──────────┬───────────┘            │
│                        ▼                         │
│               ┌─────────────────┐               │
│               │   App Pod       │               │
│               │   (envFrom)     │               │
│               └─────────────────┘               │
└──────────────────────────────────────────────────┘
```

---

## Useful Commands

```bash
# List all ConfigMaps in the namespace
kubectl get configmaps -n ivolve

# List all Secrets in the namespace
kubectl get secrets -n ivolve

# Describe a ConfigMap
kubectl describe configmap mysql-config -n ivolve

# Describe a Secret (values will be hidden)
kubectl describe secret mysql-secret -n ivolve

# Delete both objects
kubectl delete configmap mysql-config -n ivolve
kubectl delete secret mysql-secret -n ivolve
```

---

## Security Notes

- Never commit plain-text passwords to Git. Always store the base64-encoded values in the YAML.
- For production environments, consider using tools like **HashiCorp Vault**, **Sealed Secrets**, or **AWS Secrets Manager** instead of native Kubernetes Secrets.
- Kubernetes Secrets are only base64-encoded, **not encrypted**, by default. Enable [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) for stronger security.
