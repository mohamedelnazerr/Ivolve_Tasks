# Lab 16: Kubernetes Init Container for Pre-Deployment Database Setup

## What This Lab Does
Modifies the existing Node.js Deployment to include an init container that creates the `ivolve` database and user in MySQL **before** the main app starts.

---

## How It Works
```
Pod Startup Order:
[Init Container: mysql-init] → creates DB & user → [Main Container: nodejs-app starts]
```

---

## Changes Made
Modified `deployment.yml` to add an `initContainers` section using `mysql:5.7` image.

```yaml
initContainers:
  - name: mysql-init
    image: mysql:5.7
    env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: MYSQL_ROOT_PASSWORD
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-secret
            key: DB_PASSWORD
      - name: DB_HOST
        valueFrom:
          configMapKeyRef:
            name: mysql-config
            key: DB_HOST
      - name: DB_USER
        valueFrom:
          configMapKeyRef:
            name: mysql-config
            key: DB_USER
    command:
      - bash
      - -c
      - |
        echo "Waiting for MySQL to be ready..."
        until mysql -h mysql-headless -u root -p$MYSQL_ROOT_PASSWORD -e "SELECT 1;" &>/dev/null; do
          echo "MySQL not ready yet, retrying in 5s..."
          sleep 5
        done
        mysql -h mysql-headless -u root -p$MYSQL_ROOT_PASSWORD <<EOF
        CREATE DATABASE IF NOT EXISTS ivolve;
        CREATE USER IF NOT EXISTS '$DB_USER'@'%' IDENTIFIED BY '$DB_PASSWORD';
        GRANT ALL PRIVILEGES ON ivolve.* TO '$DB_USER'@'%';
        FLUSH PRIVILEGES;
        EOF
```

## Apply

```bash
kubectl apply -f deployment.yml
```

---

## Verify

```bash
# Check init container logs
kubectl logs -n ivolve -l app=nodejs-app -c mysql-init

# Connect to MySQL and verify DB and user exist
kubectl exec -it mysql-0 -n ivolve -- mysql -u root -p
```

Inside MySQL:
```sql
SHOW DATABASES;
SELECT user, host FROM mysql.user WHERE user='ivolve';
SHOW GRANTS FOR 'ivolve'@'%';
exit
```

---

## Expected Result
- `ivolve` database exists
- `ivolve` user exists with full privileges on the `ivolve` database
- Node.js app starts only after init container finishes successfully
