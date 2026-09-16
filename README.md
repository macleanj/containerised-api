# Task Manager API 🐳

> A production-grade REST API built with **FastAPI**, containerised with **Docker**, backed by **PostgreSQL** for persistence and **Redis** for caching — fully orchestrated with Docker Compose and deployed live on Railway.

[![CI](https://img.shields.io/github/actions/workflow/status/HeshMaddage/containerised-api/ci.yml?label=CI&logo=github)](https://github.com/HeshMaddage/containerised-api)
[![Docker](https://img.shields.io/badge/Docker-ready-blue?logo=docker)](https://hub.docker.com/)
[![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110-green?logo=fastapi)](https://fastapi.tiangolo.com)

---

## What this project is

This is **Project 1** of my 3-month AI/MLOps/Cloud/DevOps learning roadmap.

The goal wasn't to build a fancy task manager — it was to deeply understand **how production containerisation actually works**: multi-stage builds, service orchestration, caching layers, health checks, and deployment via Docker Hub. The Task Manager is the vehicle; Docker is the lesson.

---

## Live demo

**Base URL:** `https://taskapi-production-60e9.up.railway.app`

| Endpoint      | Method | Description                   |
|---------------|--------|-------------------------------|
| `/health`     | GET    | Service health check          |
| `/tasks`      | GET    | List all tasks (Redis cached) |
| `/tasks`      | POST   | Create a new task             |
| `/tasks/{id}` | GET    | Get a specific task           |
| `/tasks/{id}` | PUT    | Update a task                 |
| `/tasks/{id}` | DELETE | Delete a task                 |

**Interactive API docs:** `https://taskapi-production-60e9.up.railway.app/docs`

Try it right now:
```bash
# Create a task
curl -X POST https://taskapi-production-60e9.up.railway.app/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "My first task", "description": "Testing the live API"}'

# List tasks and see the cache header
curl -v https://taskapi-production-60e9.up.railway.app/tasks 2>&1 | grep -E "X-Cache|HTTP"
# First call:  X-Cache: MISS  (hits Postgres)
# Second call: X-Cache: HIT   (served from Redis)
```

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  Docker Network                 │
│                                                 │
│  ┌──────────────┐   SQL    ┌──────────────────┐ │
│  │   FastAPI    │─────────►│   PostgreSQL 15  │ │
│  │  :8000       │          │   :5432          │ │
│  │              │◄─────────│                  │ │
│  └──────┬───────┘   rows   └──────────────────┘ │
│         │                                       │
│         │  GET/SET  ┌──────────────────┐        │
│         └──────────►│   Redis 7        │        │
│                     │   :6379          │        │
│                     └──────────────────┘        │
└─────────────────────────────────────────────────┘
                    ▲
                    │ HTTP :8000
                    │
               [Client / curl]
```

**Request flow for `GET /tasks`:**
1. Client sends request to FastAPI on port 8000
2. FastAPI checks Redis for a cached response (`tasks:all` key)
3. **Cache HIT** → return instantly from Redis (~1ms), set `X-Cache: HIT` header
4. **Cache MISS** → query PostgreSQL, store result in Redis with 60s TTL, set `X-Cache: MISS`
5. On any write (POST / PUT / DELETE) → invalidate relevant Redis keys so stale data is never served

---

## Tech stack

| Tool              | Role               | Why this choice |
|-------------------|--------------------|-----------------|
| **FastAPI**       | REST API framework | Async-native, auto-generates OpenAPI docs, Pydantic validation built in |
| **PostgreSQL 15** | Persistent storage | Industry standard relational DB; alpine image keeps it lightweight |
| **Redis 7**       | Response caching   | In-memory store with TTL support; sub-millisecond reads |
| **Docker**        | Containerisation   | Reproducible environments; same image runs locally and in production |
| **Docker Compose**| Multi-container orchestration | Declares all services + network in one file; one command to run everything |
| **Railway**       | Cloud deployment   | Zero-config Docker deployments; free tier; auto-provisions Postgres + Redis |
| **SQLAlchemy**    | ORM                | Decouples DB logic from route handlers; manages connection pooling |
| **Pydantic**      | Data validation    | Request/response schema validation; auto-generates API docs |

---

## Project structure

```
project1-containerised-api/
├── app/
│   ├── __init__.py
│   ├── main.py          # FastAPI app, all route handlers
│   ├── models.py        # SQLAlchemy ORM models (DB table definitions)
│   ├── schemas.py       # Pydantic schemas (request/response shape)
│   ├── database.py      # DB engine, session factory, dependency
│   └── cache.py         # Redis client, get/set/invalidate helpers
├── tests/
│   ├── __init__.py
│   └── test_main.py     # pytest tests using SQLite in-memory (no Docker needed)
├── Dockerfile           # Multi-stage build: builder → slim final image
├── docker-compose.yml   # Orchestrates api + postgres + redis
├── .dockerignore        # Excludes .env, venv, __pycache__ from image
├── .env.example         # Template for environment variables (safe to commit)
├── .gitignore
├── requirements.txt
└── README.md
```

---

## Docker implementation — key decisions explained

### Multi-stage build

The Dockerfile uses two stages to keep the production image lean:

```dockerfile
# Stage 1: Install dependencies using a full Python image
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Copy only installed packages into a slim base
FROM python:3.11-slim AS final
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app/ ./app/
```

| | Single-stage | Multi-stage |
|-|-------------|-------------|
| Base image | `python:3.11` (~900MB) | `python:3.11-slim` (~180MB) |
| Build tools in final image | ✅ Yes (security risk) | ❌ No (stripped out) |
| Final image size | ~950MB | ~190MB |

**Why it matters:** Smaller images pull faster in CI/CD, cost less to store in a registry, and have a smaller attack surface (fewer binaries = fewer vulnerabilities).

### Non-root user

By default Docker containers run as `root`. If an attacker exploits a vulnerability in your app, they get root access inside the container. Adding a dedicated user removes that risk:

```dockerfile
RUN adduser --disabled-password --gecos "" --uid 1001 appuser
USER appuser
```

This is a hard requirement in any company with a security policy (which is most of them).

### Health checks + startup ordering

Without health checks, Docker Compose starts all services simultaneously. FastAPI tries to connect to Postgres before it's ready → crashes on boot. The health check makes the API wait until Postgres is actually accepting connections:

```yaml
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
    interval: 5s
    retries: 5

api:
  depends_on:
    postgres:
      condition: service_healthy   # waits for healthcheck to pass
```

### Named volumes for data persistence

```yaml
volumes:
  postgres_data:   # survives docker compose down
  redis_data:
```

`docker compose down` stops containers but keeps data. `docker compose down -v` wipes volumes too (useful for a clean reset during development).

### Layer caching optimisation

```dockerfile
# Copy requirements BEFORE source code
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Source code changes don't invalidate the pip cache
COPY app/ ./app/
```

If you change a source file but not `requirements.txt`, Docker reuses the cached pip layer. A rebuild goes from ~3 minutes to ~10 seconds.

---

## Run locally

**Requirements:** Docker Desktop installed and running. Nothing else.

```bash
# 1. Clone the repo
git clone https://github.com/HeshMaddage/containerised-api
cd containerised-api

# 2. Set up environment variables
cp .env.example .env
# Edit .env if needed (defaults work for local dev)

# 3. Start all three services
docker compose up --build

# API:      http://localhost:8000
# API docs: http://localhost:8000/docs
# Postgres: localhost:5432
# Redis:    localhost:6379
```

**Verify Redis caching is working:**

```bash
# First request — should see X-Cache: MISS
curl -s -D - http://localhost:8000/tasks -o /dev/null | grep X-Cache

# Second request — should see X-Cache: HIT
curl -s -D - http://localhost:8000/tasks -o /dev/null | grep X-Cache
```

**Useful commands during development:**

```bash
docker compose logs -f api         # stream FastAPI logs
docker compose logs -f postgres    # stream Postgres logs
docker compose ps                  # see status of all services
docker exec -it <api_container> bash          # shell into the API container
docker exec -it <postgres_container> psql -U taskuser taskdb  # Postgres shell
docker compose down -v             # stop everything and wipe data (clean slate)
```

---

## Run tests

Tests use SQLite in-memory — no Docker or Postgres required to run them.

```bash
pip install -r requirements.txt pytest httpx
pytest tests/ -v
```

Expected output:
```
tests/test_main.py::test_health_check    PASSED
tests/test_main.py::test_create_task     PASSED
tests/test_main.py::test_list_tasks      PASSED
tests/test_main.py::test_get_task        PASSED
tests/test_main.py::test_update_task     PASSED
tests/test_main.py::test_delete_task     PASSED
tests/test_main.py::test_task_not_found  PASSED

7 passed in 0.42s
```

The test suite overrides the database dependency with SQLite so tests run in complete isolation — no network, no Postgres, no state between runs.

---

## Deployment

The app is deployed on **Railway** using the Docker image pushed to Docker Hub.

**Deployment pipeline:**
```
Code change → git push → GitHub Actions builds image
→ pushes to Docker Hub → Railway pulls latest image → live
```

Environment variables are set in Railway's dashboard (never stored in code or committed):

```
DATABASE_URL=postgresql://...    # Railway-provisioned Postgres
REDIS_HOST=...                   # Railway-provisioned Redis
REDIS_PORT=6379
```

**To deploy your own instance:**
1. Fork this repo
2. Create a Railway account at [railway.app](https://railway.app)
3. New Project → Deploy from GitHub repo
4. Add Postgres and Redis plugins
5. Set environment variables from `.env.example`
6. Railway auto-builds from your Dockerfile on every push to `main`

---

## What I learned

### 1. Multi-stage builds cut image size by 80%

Before discovering multi-stage builds, my image was ~950MB because `python:3.11` ships with gcc, build tools, and a ton of utilities you don't need at runtime. A two-stage build — install dependencies in the full image, copy only the installed packages into `python:3.11-slim` — brought it to ~190MB. This directly reduces pull times in CI/CD, storage costs in ECR/Docker Hub, and startup time on cold deploys.

### 2. Non-root users in containers are non-negotiable in production

Running a process as root inside a container gives an attacker the keys to the container's filesystem if they exploit an app vulnerability. Adding a non-root `appuser` is a one-line fix (`USER appuser` in the Dockerfile) but it's the difference between a compliant and non-compliant production deployment at any serious company.

### 3. Docker Compose health checks prevent startup race conditions

The first time I ran `docker compose up` without health checks, FastAPI crashed on boot every time because it tried to connect to Postgres before the DB was ready. Adding a `pg_isready` health check with `depends_on: condition: service_healthy` made the API wait until Postgres was actually accepting connections. A subtle but critical detail for any multi-service stack.

### 4. Redis caching is a database query multiplier, not a workaround

I initially thought caching was just "making things faster." What I actually learned: a single Redis `GET` call costs ~1ms and zero Postgres compute. For endpoints that are read-heavy and change infrequently (like listing tasks), caching turns 50ms Postgres round-trips into 1ms Redis reads. The `X-Cache: HIT/MISS` response header makes the caching behaviour transparent — useful for debugging and for demonstrating the system to others.

### 5. Layer order in Dockerfiles is a performance contract

If you `COPY . .` before `RUN pip install`, then any code change (including a typo fix) invalidates the pip cache and reruns the full install. Moving `COPY requirements.txt .` and `RUN pip install` before copying source code means pip only reruns when `requirements.txt` changes. This turned 3-minute rebuilds into 10-second rebuilds during development.

### 6. Named volumes are the difference between stateful and stateless containers

Containers are ephemeral by design — but your data shouldn't be. `docker compose down` removes containers. Without a named volume (`postgres_data`), all task data disappears. With the volume, the data persists in Docker's managed storage and remounts when you bring the stack back up.

---

## Author

**Hesh Maddage** — CS undergraduate @ University of Sri Jayewardenepura
Building across AI Engineering · MLOps · DevOps · Cloud · Automation

---

## Build

```bash

# Locally build and pushed image (example for version 5.3)
# docker login
# docker buildx create --name multiarch --driver docker-container --use # First time docker setup
docker buildx build --push --platform linux/arm/v7,linux/arm64,linux/amd64 --tag jmaclean/fast-api:0.1 .

# docker buildx build --push --platform linux/arm/v7,linux/arm64,linux/amd64 --tag jmaclean/hello-world:6.3-http . # when https disabled in config

# Copy specific architecture
crane copy <SOURCE> <DESTINATION>

```
