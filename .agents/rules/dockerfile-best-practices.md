---
trigger: model_decision
description: Use when writing, reviewing, or generating any Dockerfile. Covers multi-stage builds, distroless final stages, selective COPY, layer cache optimization, non-root users,signal handling,and security hardening.Always consult before touching a Dockerfile
---

# Dockerfile Best Practices

This skill governs how Claude writes and reviews Dockerfiles. Read it fully before producing any Dockerfile output. These rules are non-negotiable unless the user explicitly overrides a specific item.

---

## Anti-Patterns ❌

### Service Design
- **Never run 2 services in one container** (e.g. nginx + php-fpm, app + cron). Each container should have a single responsibility. Use Docker Compose to orchestrate multiple services.

### File Copying
- **Never use `COPY . .` as a blanket copy** — it pulls in everything unless perfectly covered by `.dockerignore`, which is fragile.
- **Never copy files that don't belong in production:**
  - `.env` — secrets must not be baked in
  - `tests/`, `spec/`, `__tests__/`
  - `node_modules/` — always reinstall inside the image
  - `.git/`
  - `docker/`, `docker-compose*.yml`
  - `*.md`, `*.log`, `*.local`
  - `.github/`, `.vscode/`, `.idea/`
- **Never use `ADD` for local files** — use `COPY`. `ADD` has implicit behaviors (auto-extract tarballs, remote URL fetching) that make builds unpredictable. Reserve `ADD` only when its tar-extraction feature is explicitly needed.

### Secrets & Config
- **Never store secrets in `ENV` or `ARG`** — they are visible in `docker history` and image metadata. Use runtime secret mounts (`--mount=type=secret`) or an external secrets manager.
- **Never hardcode environment-specific values** (URLs, IPs, API endpoints, credentials) — pass them at runtime via environment variables.

### Layers & Caching
- **Never split `apt-get update` from `apt-get install`** into separate `RUN` commands — stale cache will cause `install` to fetch outdated package lists.
  ```dockerfile
  # ❌ Bad
  RUN apt-get update
  RUN apt-get install -y curl

  # ✅ Good
  RUN apt-get update && apt-get install -y --no-install-recommends curl \
      && rm -rf /var/lib/apt/lists/*
  ```
- **Never copy source code before installing dependencies** — it busts the cache on every code change.
  ```dockerfile
  # ❌ Bad
  COPY . .
  RUN npm ci

  # ✅ Good
  COPY package.json package-lock.json ./
  RUN npm ci
  COPY src/ ./src/
  ```
- **Never leave cleanup in a separate layer** — `rm`, cache purges, and temp file removal must happen in the same `RUN` command as the install.

### Base Image
- **Never use `latest` tag** — it makes builds non-reproducible and silently breaks on upstream changes.
- **Never use a full OS image** (e.g. `ubuntu`, `debian`) when alpine or distroless is sufficient.
- **Never install dev/debug tools in production images** — no `curl`, `wget`, `vim`, `bash`, `git` unless strictly required at runtime.

### Process & Runtime
- **Never use shell form for `ENTRYPOINT`** — it wraps the process in `/bin/sh -c`, which swallows signals (SIGTERM, SIGINT) and breaks graceful shutdown.
  ```dockerfile
  # ❌ Bad — shell form
  ENTRYPOINT node server.js

  # ✅ Good — exec form
  ENTRYPOINT ["node", "server.js"]
  ```
- **Never run as root** — always define and switch to a non-root user before the final `CMD`/`ENTRYPOINT`.
- **Never operate from `/`** as the working directory — always set an explicit `WORKDIR`.

---

## Best Practices ✅

### Build Strategy

**Always use multi-stage builds.**
Separate the build environment from the runtime environment. The builder stage can be fat; the final stage must be lean.

```dockerfile
FROM node:20.11-alpine AS builder
# ... install deps, build assets

FROM gcr.io/distroless/nodejs20-debian12
# ... copy only runtime artifacts
```

**Watch for cross-language build dependencies.**
When constructing multi-stage builds for full-stack apps (e.g., building a Laravel/Vite frontend in a Node builder), explicitly verify if the bundler plugins natively execute or require the backend language to resolve paths. If so, you MUST `COPY composer.json` (or equivalent manifests) into the frontend builder stage to prevent silent compilation failures.

**Use distroless or alpine as the final stage.**
- `gcr.io/distroless/*` — no shell, no package manager, minimal attack surface. Preferred for production.
- `*-alpine` — very small, has a shell if debugging is ever needed. Acceptable when distroless isn't available for the runtime.
- Never use `:debug` distroless variants in production — only in staging/dev.

**Available distroless images by language:**
| Runtime | Image |
|---|---|
| Node.js | `gcr.io/distroless/nodejs20-debian12` |
| Python | `gcr.io/distroless/python3-debian12` |
| Java | `gcr.io/distroless/java21-debian12` |
| Go / static binary | `gcr.io/distroless/static-debian12` |
| C / C++ | `gcr.io/distroless/cc-debian12` |

---

### Distroless Selective COPY Strategy

**Never use `COPY --from=builder /app .` in the final stage.** This copies everything from the builder — including source, dev tools, tests, and build artifacts — defeating the entire purpose of multi-stage builds.

Instead, **enumerate directories explicitly** — copy only what the application needs to run.

**Generic pattern:**
```dockerfile
# Only copy what runs, not what builds
COPY --from=builder /app/dist        /app/dist
COPY --from=builder /app/node_modules /app/node_modules
```

**Laravel (PHP) example:**
```dockerfile
COPY --from=builder /app/app         /var/www/html/app
COPY --from=builder /app/bootstrap   /var/www/hdoctml/bootstrap
COPY --from=builder /app/config      /var/www/html/config
COPY --from=builder /app/database    /var/www/html/database
COPY --from=builder /app/public      /var/www/html/public
COPY --from=builder /app/resources   /var/www/html/resources
COPY --from=builder /app/routes      /var/www/html/routes
COPY --from=builder /app/storage     /var/www/html/storage
COPY --from=builder /app/vendor      /var/www/html/vendor
COPY --from=builder /app/artisan     /var/www/html/artisan
# Excluded: tests/, .env, node_modules/, .git/, docker/, *.md, webpack.mix.js, phpunit.xml
```

**Node.js example:**
```dockerfile
COPY --from=builder /app/dist        /app/dist
COPY --from=builder /app/node_modules /app/node_modules
COPY --from=builder /app/package.json /app/package.json
# Excluded: src/, tests/, .env, *.config.js, tsconfig.json
```

**Go example (static binary — no runtime deps needed):**
```dockerfile
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /app/server
```

---

### Base Image

- **Pin exact versions:** `node:20.11-alpine` not `node:20-alpine` not `node:latest`
- **Pin OS package versions** when installing:
  ```dockerfile
  RUN apt-get install -y --no-install-recommends curl=7.88.1-10+deb12u5
  ```
- **Prefer slim/alpine variants** for builder stages when you don't need distroless in the final stage.

---

### Layer & Cache Optimization

Order layers from **least → most frequently changed**:
1. Base image
2. System dependencies (rarely change)
3. Dependency manifests (`package.json`, `composer.json`, `requirements.txt`)
4. Dependency install (`npm ci`, `composer install`)
5. Application source code (changes most often)
6. Build step

```dockerfile
FROM node:20.11-alpine AS builder
WORKDIR /app

# System deps (rarely changes)
RUN apk add --no-cache python3 make g++

# Dependency manifests (changes less often than source)
COPY package.json package-lock.json ./

# Install (cached unless manifests change)
RUN npm ci --only=production

# Source (changes most often — last)
COPY src/ ./src/

RUN npm run build
```

**Cleanup rules:**
- `apt`: always append `&& rm -rf /var/lib/apt/lists/*`
- `apk`: use `--no-cache` flag — no cleanup needed
- `pip`: use `--no-cache-dir`
- `npm`: use `npm ci` (not `npm install`) for reproducible installs, then run `npm audit fix` to patch known vulnerabilities at build time:
  ```dockerfile
  RUN npm ci && npm audit fix
  ```
  If audit fails the build (exit non-zero), do not suppress it with `--force` unless you have explicitly reviewed the advisory — a failing audit is a signal, not noise.

---

### Security

**Non-root user — always.**
```dockerfile
# Alpine / Debian
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

# Distroless — use built-in nonroot user
USER nonroot

# Or by UID (preferred in Kubernetes environments)
USER 1001
```

**Secret mounts — never `ENV` or `ARG` for secrets.**
```dockerfile
# ✅ Build-time secret (not stored in image layers)
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci
```

**Filesystem permissions:**
- Ensure the non-root user has write access only to directories it needs (`/tmp`, `/app/storage`)
- Make application source files read-only where possible

---

### Process & Runtime

**Always use exec form for `ENTRYPOINT` and `CMD`:**
```dockerfile
ENTRYPOINT ["node", "dist/server.js"]
CMD ["--port", "3000"]
```

**Use an init process for signal handling and zombie reaping:**
```dockerfile
# Option 1 — tini (explicit)
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--", "node", "server.js"]

# Option 2 — Docker's built-in init
# docker run --init ...

# Option 3 — dumb-init
COPY --from=builder /usr/bin/dumb-init /usr/bin/dumb-init
ENTRYPOINT ["dumb-init", "--", "node", "server.js"]
```

**Always define `HEALTHCHECK`:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```
For distroless images (no `curl`), use `wget` or a compiled health binary, or rely on the orchestrator (Kubernetes liveness probe).

**Always set `WORKDIR`:**
```dockerfile
WORKDIR /app   # Never rely on default /
```

**Only `EXPOSE` ports actually used:**
```dockerfile
EXPOSE 3000   # Document, don't rely on — actual port binding is at runtime
```

---

### `.dockerignore`

Always include a `.dockerignore`. At minimum:
```
.git/
.gitignore
.env*
*.md
node_modules/
vendor/
tests/
__tests__/
spec/
.github/
.vscode/
docker/
docker-compose*.yml
Makefile
*.log
coverage/
dist/      # only if not needed in builder context
```

---

### Metadata

Label every image for traceability:
```dockerfile
LABEL maintainer="team@example.com" \
      version="1.0.0" \
      description="Production API server" \
      org.opencontainers.image.source="https://github.com/org/repo"
```

---

## Complete Example (Node.js)

```dockerfile
# ── Stage 1: Builder ─────────────────────────────────────────────────────────
FROM node:20.11-alpine AS builder

WORKDIR /app

# System build deps
RUN apk add --no-cache python3 make g++

# Install deps (cached layer)
COPY package.json package-lock.json ./
RUN npm ci

# Copy source and build
COPY src/ ./src/
COPY tsconfig.json ./
RUN npm run build && npm prune --production

# ── Stage 2: Runtime (distroless) ────────────────────────────────────────────
FROM gcr.io/distroless/nodejs20-debian12

LABEL maintainer="team@example.com" \
      version="1.0.0"

WORKDIR /app

# Selective copy — only runtime artifacts
COPY --from=builder /app/dist        ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json

USER nonroot

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD ["/nodejs/bin/node", "-e", "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"]

ENTRYPOINT ["node", "dist/server.js"]
```

---

## Quick Checklist Before Finalizing Any Dockerfile

- [ ] Multi-stage build used?
- [ ] Final stage is distroless or alpine (not full OS)?
- [ ] Base image pinned to exact version?
- [ ] Dependencies installed before source copied?
- [ ] Selective COPY in final stage (no `COPY --from=builder /app .`)?
- [ ] `.dockerignore` exists and covers `.env`, `tests/`, `.git/`, `node_modules/`?
- [ ] No secrets in `ENV` or `ARG`?
- [ ] Non-root user set?
- [ ] Exec form used for `ENTRYPOINT`/`CMD`?
- [ ] `WORKDIR` explicitly set?
- [ ] `HEALTHCHECK` defined?
- [ ] Cleanup in same `RUN` layer as install?
- [ ] Only one service per container?