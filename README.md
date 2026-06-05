# 🐳 Podman Pod PoC — PostgreSQL + pgAdmin on WSL

A hands-on Proof of Concept demonstrating **Podman Pods** using PostgreSQL and pgAdmin — running inside a shared Pod on WSL, communicating via `localhost` exactly like Kubernetes Pods.

---

## 📐 Architecture

```
┌─────────────────────────────────────────────────┐
│               postgres-pod                      │
│         (Shared Network Namespace)              │
│                                                 │
│  ┌─────────────┐         ┌──────────────────┐   │
│  │ postgres-db │◄───────►│    pgadmin       │   │
│  │  port 5432  │localhost│    port 80       │   │
│  └─────────────┘         └──────────────────┘   │
│         │                        │              │
│  ┌──────────────────────────────────────────┐   │
│  │          Infra (pause) container         │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
         │                        │
    Host :5432               Host :8080
```

---

## 🎯 PoC Objectives

- Run multiple containers as a single managed unit (Pod)
- Demonstrate shared network namespace (`localhost` between containers)
- Show pod-level lifecycle (start/stop all together)
- Generate Kubernetes-compatible YAML from a running Pod
- Redeploy from YAML using `podman play kube`

---

## ✅ Prerequisites

- WSL (Ubuntu 22.04+)
- Podman v5.x

```bash
# Install Podman if not present
sudo apt update && sudo apt install -y podman

# Verify
podman --version
```

---

## 🚀 Step-by-Step Setup

### 1. Pull Images

```bash
podman pull docker.io/library/postgres:17
podman pull docker.io/dpage/pgadmin4:latest

podman images   # verify both present
```

---

### 2. Create the Pod

```bash
podman pod create \
  --name postgres-pod \
  -p 5432:5432 \
  -p 8080:80

podman pod ps   # expect STATUS: Created
```

---

### 3. Start PostgreSQL Container

```bash
podman run -d \
  --name postgres-db \
  --pod postgres-pod \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres123 \
  -e POSTGRES_DB=testdb \
  postgres:17

podman ps   # verify Up
```

---

### 4. Start pgAdmin Container

```bash
podman run -d \
  --name pgadmin \
  --pod postgres-pod \
  -e PGADMIN_DEFAULT_EMAIL=admin@test.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin123 \
  dpage/pgadmin4

podman ps   # both containers must show Up
```

---

### 5. Create Test Data

```bash
podman exec -it postgres-db psql -U postgres -d testdb
```

Inside `psql`:

```sql
CREATE TABLE employees (
    emp_id     SERIAL PRIMARY KEY,
    emp_name   VARCHAR(100),
    department VARCHAR(50)
);

INSERT INTO employees (emp_name, department) VALUES
    ('John',  'DBA'),
    ('Mary',  'Developer'),
    ('David', 'DevOps');

SELECT * FROM employees;
\q
```

---

### 6. Validate Pod Networking

Prove both containers share `localhost` by connecting to PostgreSQL **from inside the pgAdmin container**:

```bash
# Enter pgAdmin container as root (Alpine-based, use apk + sh)
podman exec -it --user root pgadmin sh

apk add --no-cache postgresql-client

psql -h localhost -U postgres -d testdb -c "SELECT count(*) FROM employees;"
exit
```

**Expected output:** `count = 3`

> 💡 `localhost` works because both containers share the Pod's network namespace — no container IPs needed.

---

### 7. Access pgAdmin UI

Open in browser: **http://localhost:8080**

| Field    | Value            |
|----------|-----------------|
| Email    | admin@test.com   |
| Password | admin123         |

**Add PostgreSQL Server → Connection tab:**

| Field    | Value        |
|----------|-------------|
| Host     | localhost    |
| Port     | 5432         |
| Database | testdb       |
| Username | postgres     |
| Password | postgres123  |

---

### 8. Pod Lifecycle Demo

```bash
# Stop all containers in the pod together
podman pod stop postgres-pod

podman ps   # no running containers

# Start all containers in the pod together
podman pod start postgres-pod

podman ps   # both back up

# Inspect and logs
podman pod inspect postgres-pod
podman logs postgres-db
podman logs pgadmin
```

---

### 9. Generate Kubernetes YAML

```bash
podman generate kube postgres-pod > postgres-pod.yaml
cat postgres-pod.yaml
```

---

### 10. Redeploy from YAML

```bash
# Tear down
podman pod stop postgres-pod && podman pod rm -f postgres-pod

# Redeploy from YAML
podman play kube postgres-pod.yaml

podman pod ps
podman ps
```

> ⚠️ **Note:** After `podman play kube`, container names are prefixed with the pod name:
> - `postgres-db` → `postgres-pod-postgres-db`
> - `pgadmin`     → `postgres-pod-pgadmin`
>

```bash
# Use updated container name
podman exec -it postgres-pod-postgres-db psql -U postgres -d testdb
SELECT * FROM employees;
\q
```

---

## ✅ Validation Checklist

| Check                    | Command                                                        |
|--------------------------|----------------------------------------------------------------|
| Pod running              | `podman pod ps`                                                |
| Containers up            | `podman ps`                                                    |
| DB accessible            | `podman exec -it postgres-db psql -U postgres -d testdb`       |
| Test data present        | `SELECT * FROM employees;`                                     |
| pgAdmin UI               | http://localhost:8080                                          |
| Shared networking        | `psql -h localhost` from inside pgadmin container              |
| Kube YAML generated      | `podman generate kube postgres-pod`                            |
| Redeployed from YAML     | `podman play kube postgres-pod.yaml`                           |

---

## 🧹 Cleanup

```bash
podman pod stop postgres-pod
podman pod rm -f postgres-pod
podman rm -f postgres-db pgadmin 2>/dev/null || true

podman rmi postgres:17
podman rmi dpage/pgadmin4

# Verify clean
podman pod ps -a
podman ps -a
podman images
```

---

## 📚 Key Learnings

| Concept                     | Takeaway                                                         |
|-----------------------------|------------------------------------------------------------------|
| Pod = unit of containers    | Multiple containers managed as one                               |
| Shared network namespace    | Containers talk via `localhost`, no IPs needed                   |
| Infra container             | Pause container holds the network namespace for the pod          |
| Pod-level lifecycle         | `pod stop` / `pod start` controls all containers together        |
| Kubernetes YAML generation  | `podman generate kube` produces deployable K8s manifests         |
| `podman play kube`          | Redeploy from YAML — names get pod-prefixed, verify ports        |
| Alpine vs Debian images     | pgAdmin is Alpine-based → use `apk`, not `apt`; use `sh` not `bash` |

---


## 🔗 References

- [Podman Official Docs](https://docs.podman.io/)
- [pgAdmin Docker Image](https://hub.docker.com/r/dpage/pgadmin4)
- [PostgreSQL Docker Image](https://hub.docker.com/_/postgres)
- [Podman Pod man page](https://docs.podman.io/en/latest/markdown/podman-pod.1.html)
