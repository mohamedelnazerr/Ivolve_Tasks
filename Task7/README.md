# 🐳 Lab 7: Docker Volume and Bind Mount with Nginx

## 🎯 Objective

Use Docker **named volumes** to persist Nginx logs and **bind mounts** to serve a custom HTML page from the host machine.

---

## 📁 Directory Structure

```
nginx-bind/
└── html/
    └── index.html
```

---

## Prerequisites

- Docker installed and running
- Basic knowledge of the Linux command line

---

## 🚀 Steps

### 1. Create the `nginx_logs` Named Volume

```bash
docker volume create nginx_logs
```

Verify it was created and note its mount path on the host:

```bash
docker volume inspect nginx_logs
```

> The default path is `/var/lib/docker/volumes/nginx_logs/_data`

---

### 2. Create the Bind Mount Directory and HTML File

```bash
mkdir -p ~/nginx-bind/html
echo "<h1>Hello from Bind Mount</h1>" > ~/nginx-bind/html/index.html
```

---

### 3. Run the Nginx Container

```bash
docker run -d \
  --name nginx-lab7 \
  -p 8080:80 \
  -v nginx_logs:/var/log/nginx \
  -v ~/nginx-bind/html:/usr/share/nginx/html \
  nginx
```

| Flag | Purpose |
|------|---------|
| `-v nginx_logs:/var/log/nginx` | Named volume → persists Nginx logs |
| `-v ~/nginx-bind/html:/usr/share/nginx/html` | Bind mount → serves custom HTML |
| `-p 8080:80` | Maps host port 8080 to container port 80 |

---

### 4. Verify the Nginx Page

```bash
curl http://localhost:8080
```

**Expected output:**

```html
<h1>Hello from Bind Mount</h1>
```

---

### 5. Modify the HTML and Verify Live Update

```bash
echo "<h1>Updated from Bind Mount!</h1>" > ~/nginx-bind/html/index.html
curl http://localhost:8080
```

> ✅ Changes reflect **instantly** — no container restart needed!

---

### 6. Verify Logs Are Stored in the Volume

```bash
# From inside the container
docker exec nginx-lab7 ls /var/log/nginx

# From the host
sudo ls /var/lib/docker/volumes/nginx_logs/_data
```

You should see `access.log` and `error.log` files.

---

### 7. Cleanup

```bash
docker stop nginx-lab7
docker rm nginx-lab7
docker volume rm nginx_logs
```

---

## 📌 Key Concepts

| Concept | Description |
|---------|-------------|
| **Named Volume** | Managed by Docker; persists data independently of container lifecycle |
| **Bind Mount** | Maps a host directory directly into a container; changes sync in real time |

---

> 🏫 Part of the **iVolve Containerization with Docker** training track.
