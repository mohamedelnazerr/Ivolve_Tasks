# 🐳 Lab 8: Custom Docker Network for Microservices

## 🎯 Objective

Build a two-service Python/Flask application (frontend + backend), connect them via a **custom Docker network**, and verify that containers on the same network can communicate while isolated containers cannot.

---

## 📁 Project Structure

```
Docker5/
├── frontend/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
└── backend/
    ├── app.py
    └── Dockerfile
```

---

## Prerequisites

- Docker installed and running
- Git installed
- Basic knowledge of Python and Flask

---

## 🚀 Steps

### 1. Clone the Repository

```bash
git clone https://github.com/Ibrahim-Adel15/Docker5.git
cd Docker5
```

---

### 2. Write the Frontend Dockerfile

Create `frontend/Dockerfile`:

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

Build the frontend image:

```bash
docker build -t frontend-img ./frontend
```

---

### 3. Write the Backend Dockerfile

Create `backend/Dockerfile`:

```dockerfile
FROM python:3.9-slim
WORKDIR /app
RUN pip install flask
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

Build the backend image:

```bash
docker build -t backend-img ./backend
```

---

### 4. Create the Custom Network

```bash
docker network create ivolve-network
```

---

### 5. Run the Containers

```bash
# Backend — on ivolve-network
docker run -d \
  --name backend \
  --network ivolve-network \
  backend-img

# Frontend1 — on ivolve-network (can reach backend)
docker run -d \
  --name frontend1 \
  --network ivolve-network \
  -p 5001:5000 \
  frontend-img

# Frontend2 — on default network (isolated from backend)
docker run -d \
  --name frontend2 \
  -p 5002:5000 \
  frontend-img
```

---

### 6. Verify Container Communication

```bash
# ✅ frontend1 CAN reach backend (same network — DNS by container name)
docker exec frontend1 ping backend

# ❌ frontend2 CANNOT reach backend (different network)
docker exec frontend2 ping backend
```

> Docker's built-in DNS resolves container names automatically within the same custom network.

---

### 7. Cleanup

```bash
docker stop frontend1 frontend2 backend
docker rm frontend1 frontend2 backend
docker network rm ivolve-network
```

---

## 📌 Key Concepts

| Concept | Description |
|---------|-------------|
| **Custom Network** | Containers on the same custom network communicate by name via Docker DNS |
| **Default (bridge) Network** | Containers are isolated from custom networks; no cross-network DNS |
| **Network Isolation** | `frontend2` on the default network cannot reach `backend` on `ivolve-network` |

---

> 🏫 Part of the **iVolve Containerization with Docker** training track.
