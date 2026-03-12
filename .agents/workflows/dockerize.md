---
description: Dockerize an application end-to-end. Analyzes the app stack, then creates .dockerignore, Dockerfile, compose.yml, dev/prod overlays, and .env.example — then validates the build.
---

# Dockerize an Application

## Step 1 — Analyze the Application

Inspect the project before writing any files:

- Identify the runtime, framework, and package manager
- Read the dependency manifest to find the build command and start command
- Identify external services the app depends on (databases, caches, queues)
- Identify the port(s) the app listens on
- Check if `Dockerfile`, `compose.yml`, or `.dockerignore` already exist — review before overwriting

Summarize before proceeding:
> "This is a [runtime] app using [framework], on port [X], depending on [services]. Build: [cmd]. Start: [cmd]."

---

## Step 2 — Create `.dockerignore`

Create `.dockerignore` at the project root.

---

## Step 3 — Write the `Dockerfile`

Write a multi-stage `Dockerfile` for the stack identified in Step 1.

---

## Step 4 — Write `compose.yml`

Create the base `compose.yml`. Include all services identified in Step 1 with appropriate health checks, named networks, named volumes, and secrets configuration.

---

## Step 5 — Write `compose.override.yml`

Create the dev overlay with hot-reload bind mounts, exposed DB ports for local tooling, and dev-only services under `profiles: [dev]`.

---

## Step 6 — Write `compose.prod.yml`

Create the production overlay with resource limits, `APP_ENV=production`, external secrets, and no bind mounts.

---

## Step 7 — Create `.env.example`

Create `.env.example` with all required variables using empty or placeholder values. Confirm `.env` is in `.gitignore`.

---

## Step 8 — Validate

```bash
# Build and inspect
docker build -t app:local .
docker images app:local
docker history app:local

# Start full dev stack and verify
docker compose up --build
docker compose ps
docker compose logs -f
```

Fix any errors before closing.