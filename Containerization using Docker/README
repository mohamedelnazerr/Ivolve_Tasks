# Lab 9: Containerized Node.js and MySQL Stack Using Docker Compose

## Overview

This lab containerizes a Node.js application alongside a MySQL database using Docker Compose. The app requires a MySQL connection and depends on a database named **ivolve** to start working.

---

## Prerequisites

- Docker & Docker Compose installed
- DockerHub account
- Git installed

---

## Project Structure

```
kubernets-app/
├── Dockerfile
├── docker-compose.yml
└── app/
    └── ...
```

---

## Steps

### Step 1 — Clone the Application

```bash
git clone https://github.com/Ibrahim-Adel15/kubernets-app.git
cd kubernets-app
```

---

### Step 2 — Create docker-compose.yml

Create a `docker-compose.yml` file in the root of the project with the following content:

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=db
      - DB_USER=root
      - DB_PASSWORD=rootpassword
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=ivolve
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

> **Note:** `DB_HOST=db` works because Docker Compose places both services on the same internal network and resolves service names as hostnames automatically.

---

### Step 3 — Build and Start the Stack

```bash
docker compose up --build -d
```

Verify both containers are running:

```bash
docker compose ps
```

> **Tip:** Wait ~15 seconds for MySQL to finish initializing before testing the app. If the app exits immediately, run `docker compose restart app`.

---

### Step 4 — Verify the Application

**Main app:**
```bash
curl http://localhost:3000
```

**Health endpoint:**
```bash
curl http://localhost:3000/health
# Expected: {"status":"ok"}
```

**Readiness endpoint:**
```bash
curl http://localhost:3000/ready
# Expected: {"ready":true}
```

**Access logs inside the container:**
```bash
docker compose exec app ls /app/logs/
docker compose exec app cat /app/logs/access.log
```

---

### Step 5 — Push the Image to DockerHub

```bash
# Log in to DockerHub
docker login

# Find your built image name
docker images

# Tag it with your DockerHub username
docker tag kubernets-app-app <YOUR_DOCKERHUB_USERNAME>/ivolve-app:latest

# Push the image
docker push <YOUR_DOCKERHUB_USERNAME>/ivolve-app:latest
```

Verify the push at:
```
https://hub.docker.com/r/<YOUR_DOCKERHUB_USERNAME>/ivolve-app
```

---

## Architecture

```
┌─────────────────────────────────────┐
│         Docker Compose Network      │
│                                     │
│  ┌────────────┐    ┌─────────────┐  │
│  │  app       │───▶│  db         │  │
│  │  Node.js   │    │  MySQL 8.0  │  │
│  │  :3000     │    │             │  │
│  └────────────┘    └──────┬──────┘  │
│                           │         │
└───────────────────────────┼─────────┘
                            │
                     ┌──────▼──────┐
                     │  db_data    │
                     │  (volume)   │
                     └─────────────┘
```

---

## Environment Variables

| Variable            | Service | Value          | Description                    |
|---------------------|---------|----------------|--------------------------------|
| `DB_HOST`           | app     | `db`           | MySQL service hostname         |
| `DB_USER`           | app     | `root`         | Database user                  |
| `DB_PASSWORD`       | app     | `rootpassword` | Database password              |
| `MYSQL_ROOT_PASSWORD` | db    | `rootpassword` | MySQL root password            |
| `MYSQL_DATABASE`    | db      | `ivolve`       | Database created on startup    |

---

## Useful Commands

```bash
# View logs
docker compose logs app
docker compose logs db

# Stop the stack
docker compose down

# Stop and remove volumes
docker compose down -v

# Restart a single service
docker compose restart app
```
