# Coding Agent Prompt: Generate Full Project Context (CONTEXT.md)

---

## ROLE & OBJECTIVE

You are a senior software architect and DevOps engineer performing a
**complete project discovery**. Your sole task is to read this codebase
thoroughly and produce a single, accurate `CONTEXT.md` file that will
serve as the permanent knowledge base for all future AI agent sessions
on this project.

This file will be used as:
- The system prompt / always-applied rule in Cursor, Claude Projects,
  or any AI coding agent
- The input context for specialized agent skills
  (performance audits, security reviews, deployment checks, etc.)
- Onboarding documentation for new engineers

**Accuracy over completeness.** Only write what you can verify from the
codebase. Never guess or assume. If something is unclear, write it as a
question under the "Open Questions" section rather than stating it as fact.

---

## PHASE 1 — DISCOVERY

Read the following files and directories in order. Extract structured
facts from each. Do not skip any step.

### 1.1 Project Identity

Read:
- `README.md`, `README` (any format)
- `composer.json` → `name`, `description`, `keywords`
- `package.json` → `name`, `description`
- Any `docs/` folder

Extract:
- What does this application do? (one sentence)
- Who are the end users?
- Is it an API, a web app, a CLI tool, or a hybrid?
- Is it public-facing or internal?

---

### 1.2 Language & Framework

Read:
- `composer.json` → `require`, `require-dev`, `platform`
- `package.json` → `dependencies`, `devDependencies`
- `.php-version`, `.nvmrc`, `.tool-versions`
- `artisan` (presence = Laravel), `bin/console` (presence = Symfony)
- `config/app.php` → `version`, `name`, `env`, `timezone`, `locale`

Extract:
- Language + version (PHP 8.3, Node 20, etc.)
- Framework + version (Laravel 11, Symfony 7, etc.)
- Frontend framework (Vue 3, React 18, Inertia.js, Livewire, Blade-only)
- Build tool (Vite, Webpack, Mix)
- Key packages that affect architecture
  (Sanctum, Passport, Octane, Horizon, Telescope, Scout, Socialite,
   Spatie permission, Filament, etc.)

---

### 1.3 Database Layer

Read:
- `config/database.php`
- `.env.example`
- `docker-compose.yml` (DB service definitions)
- `database/migrations/` (all migration files — scan for patterns)

Extract:
- Database engine(s) and version (MySQL 8.0, PostgreSQL 16, SQLite, etc.)
- Number of migrations
- List of all tables (from migration filenames + Schema::create calls)
- Tables that appear to be high-volume
  (orders, logs, events, audit, notifications, sessions, jobs —
   detect by name heuristics)
- Whether soft deletes are used (`->softDeletes()` grep)
- Whether UUIDs are used as primary keys (`->uuid()` grep)
- Whether any table uses full-text indexes (`->fullText()` grep)
- Read replica configuration (multiple `read`/`write` keys in config)
- Any multi-database setup (multiple connections defined)

---

### 1.4 Cache Layer

Read:
- `config/cache.php`
- `.env.example` → `CACHE_DRIVER`, `REDIS_*`
- `docker-compose.yml` → Redis service definition

Grep codebase for:
```
Cache::, cache(, Redis::, \Illuminate\Support\Facades\Cache
```

Extract:
- Cache driver (redis, file, memcached, dynamodb, array)
- Redis version if detectable
- Whether cache tags are used (`Cache::tags(`)
- List of all cache key patterns found (e.g. `user:{id}`, `report:monthly`)
- TTL patterns in use (what durations appear)
- Whether any cache prefix is configured (`config/cache.php → prefix`)
- Whether a separate Redis database is used for cache vs sessions vs queues

---

### 1.5 Queue & Jobs Layer

Read:
- `config/queue.php`
- `config/horizon.php` (if present)
- `app/Jobs/` (all job class files)
- `app/Console/Kernel.php` → scheduled commands

Extract:
- Queue driver (redis, sqs, database, sync, beanstalkd)
- Queue names defined (default, emails, reports, critical, etc.)
- List of all Job classes with:
  - Queue they run on
  - Whether they implement `SerializesModels`
  - Whether they have `$tries`, `$backoff`, `$timeout` defined
  - Whether they have a `failed()` method
- List of all scheduled commands with their cron expression
- Whether Horizon is installed and configured

---

### 1.6 Application Architecture

Read:
- `app/` directory structure
- `app/Http/Controllers/` (list all controllers)
- `app/Models/` (list all models)
- `app/Services/` (if exists)
- `app/Repositories/` (if exists)
- `app/Actions/` (if exists — Laravel Actions pattern)
- `app/Observers/` (if exists)
- `app/Events/` + `app/Listeners/`
- `app/Providers/` (all service providers)
- `routes/web.php`, `routes/api.php`, `routes/console.php`

Extract:
- Architecture pattern in use
  (MVC only, Service layer, Repository pattern, CQRS, Action pattern)
- Number of routes (web + API separately)
- List of all Eloquent models
- For each model: table name, key relationships
  (`hasMany`, `belongsTo`, `belongsToMany`, `hasManyThrough`)
- Whether API resources are used (`app/Http/Resources/`)
- Whether Form Requests are used (`app/Http/Requests/`)
- Whether Observers are registered — which models have them
- Whether Events/Listeners are used for side effects

---

### 1.7 Authentication & Authorization

Read:
- `config/auth.php`
- `app/Http/Middleware/` (auth-related middleware)
- Check for: Sanctum, Passport, Jetstream, Breeze, Fortify, custom auth

Extract:
- Auth mechanism (session-based, token/Sanctum, OAuth/Passport, custom)
- Guard configuration (web, api, etc.)
- Whether roles/permissions are used (Spatie Permission, custom pivot, etc.)
- Whether multi-tenancy is implemented (detect by `tenant_id` columns,
  scoped queries, or packages like `stancl/tenancy`)

---

### 1.8 HTTP Server & Infrastructure

Read:
- `Caddyfile`, `caddy/`, `config/caddy/`
- `nginx.conf`, `nginx/`, `config/nginx/`
- `Dockerfile`, `docker-compose.yml`
- `.github/workflows/`, `Jenkinsfile`, `Makefile`
- `deploy.sh`, `scripts/deploy*`
- Any `ansible/` or `terraform/` directory

Extract:
- HTTP server (Caddy, Nginx, Apache, FrankenPHP)
- Whether compression is enabled (gzip, brotli, zstd)
- Whether static asset cache headers are set
- Whether HTTPS is enforced
- Container orchestration (Docker Compose, Kubernetes, ECS)
- Cloud provider and services (AWS EC2, RDS, ElastiCache, S3, SQS, etc.)
- CI/CD tool (Jenkins, GitHub Actions, GitLab CI, etc.)
- Deployment strategy (rolling, blue-green, in-place)
- Whether `config:cache`, `route:cache`, `view:cache` are run on deploy
- Whether OPcache is configured in the PHP container

---

### 1.9 Observability

Read:
- `config/logging.php`
- Check for: Sentry, Datadog, New Relic, Flare, Bugsnag packages
- Check for: Prometheus exporter, Grafana, Loki, Promtail in docker-compose
- `config/telescope.php` (if present)

Extract:
- Log driver and channels (stack, single, daily, stderr)
- Error tracking service (Sentry, Flare, etc.) — if configured
- APM or metrics stack (Prometheus + Grafana, Datadog, etc.)
- Whether Laravel Telescope is installed
- Log retention policy if detectable

---

### 1.10 Testing

Read:
- `phpunit.xml`, `phpunit.xml.dist`, `pest.config.php`
- `tests/` directory structure
- `composer.json` → test scripts

Extract:
- Testing framework (PHPUnit, Pest)
- Test structure (Feature vs Unit split)
- Approximate test count (count test files)
- Whether database testing uses `RefreshDatabase` or `DatabaseTransactions`
- Whether factories exist for all models (`database/factories/`)
- Test coverage reporting if configured

---

### 1.11 Environment & Secrets Pattern

Read:
- `.env.example` (never `.env`)
- Any `config/*.php` files for env variable usage

Extract:
- All environment variables defined in `.env.example`
  (grouped by: App, Database, Cache, Queue, Mail, Storage, Third-party)
- Third-party services integrated
  (payment gateways, SMS, email providers, storage, OAuth providers)
- Whether secret management is handled
  (AWS Secrets Manager, Vault, plain env files)

---

### 1.12 Known Issues & Technical Debt

Read:
- `TODO`, `FIXME`, `HACK`, `PERF`, `XXX` comments — grep codebase
- Any `CHANGELOG.md` or `KNOWN_ISSUES.md`
- Open `@deprecated` annotations

Extract:
- List of all `TODO`/`FIXME` comments with file and line
- Any `@deprecated` methods still in use
- Any obviously problematic patterns (magic numbers, hardcoded credentials
  in non-.env files, commented-out code blocks > 20 lines)

---

## PHASE 2 — GENERATE CONTEXT.md

Using everything discovered in Phase 1, generate the following document.
Follow the exact structure. Use only facts confirmed from the codebase.
Mark any field you could not confirm with `[UNCONFIRMED — verify manually]`.

---

```markdown
# CONTEXT.md
# Auto-generated by: context-generator agent
# Generated on: {DATE}
# Generator version: 1.0
# Last updated: {DATE}
# Update this file when: stack changes, major refactors, new services added

---

## 1. Project Overview

**Name:** {project name}
**Purpose:** {one-sentence description}
**Type:** {Web App | API | CLI | Hybrid}
**Audience:** {Internal tool | Public-facing | Both}
**Base URL:** {from .env.example APP_URL or README}

---

## 2. Technology Stack

### Backend
| Layer | Technology | Version |
|---|---|---|
| Language | PHP | {x.x} |
| Framework | Laravel | {xx} |
| HTTP Server | Caddy / Nginx | {x.x} |
| PHP Runtime | FPM / Octane / FrankenPHP | {x.x} |

### Frontend
| Layer | Technology | Version |
|---|---|---|
| Framework | Vue / React / Inertia / Blade | {x.x} |
| Build tool | Vite / Mix | {x.x} |
| CSS | Tailwind / Bootstrap / Custom | {x.x} |

### Data Layer
| Layer | Technology | Version | Notes |
|---|---|---|---|
| Primary DB | MySQL / PostgreSQL | {x.x} | {RDS / self-hosted} |
| Cache | Redis | {x.x} | {ElastiCache / self-hosted} |
| Sessions | Redis / Database / File | — | — |
| Queue backend | Redis / SQS / Database | — | — |
| Search | Meilisearch / Elasticsearch / DB FTS | — | — |
| Storage | S3 / Local / Minio | — | — |

### Infrastructure
| Layer | Technology | Notes |
|---|---|---|
| Container | Docker + Docker Compose | {version} |
| Cloud | AWS EC2 {instance type} | {region} |
| CDN / Proxy | Cloudflare Tunnel | — |
| CI/CD | Jenkins / GitHub Actions | — |
| Secrets | .env / AWS Secrets Manager | — |

---

## 3. Key Packages

### Production Dependencies (performance/architecture relevant)
{list only packages that affect how the agent should reason about the code}
- `laravel/horizon` — Queue monitoring dashboard, config at config/horizon.php
- `laravel/sanctum` — API token auth, guards: api
- `spatie/laravel-permission` — RBAC, roles/permissions on User model
- `laravel/scout` + `meilisearch/meilisearch-php` — Full-text search
- {add others found}

### Dev Dependencies (tooling relevant)
- `laravel/telescope` — Request/query introspection (dev/staging only)
- `barryvdh/laravel-debugbar` — Debug toolbar
- {add others found}

---

## 4. Database

### Tables ({N} total)

**Core tables:**
| Table | Estimated Volume | Notes |
|---|---|---|
| users | Medium | Has soft deletes |
| employees | High | FK to departments, contracts |
| departments | Low | Lookup table |
| attendance_records | Very high | ⚠️ Off-limits — DBA owns schema |
| {table} | {volume} | {notes} |

**High-volume / sensitive tables:**
{list tables that need extra care}

### Conventions
- Primary keys: {auto-increment INT | UUID | ULID}
- Soft deletes used: {Yes / No / Partial — list models}
- Timestamps: {all tables have created_at/updated_at | exceptions noted}
- FK indexes: {all FKs indexed by convention | gaps noted}

### Multi-database
{Yes — connections: mysql (primary), pgsql (reporting) | No}

### Read Replicas
{Yes — configured in config/database.php sticky=true | No | Planned}

---

## 5. Cache Architecture

### Driver
```
CACHE_DRIVER=redis
REDIS_HOST=redis
REDIS_DB=1          # cache uses DB 1, sessions DB 2, queues DB 0
```

### Cache Key Registry
| Key Pattern | TTL | Jitter | Invalidated by | Notes |
|---|---|---|---|---|
| `employee:{id}` | 300s | +rand(0,60) | EmployeeObserver | — |
| `dashboard:stats` | 300s | +rand(0,60) | EmployeeObserver | — |
| `report:monthly:{year}-{month}` | 3600s | +rand(0,300) | Scheduled refresh | High stampede risk |
| `dept:{id}:employees` | 600s | +rand(0,120) | DepartmentObserver | — |
| {add all found} | | | | |

### Cache Tag Groups
| Tag | Contains | Flush trigger |
|---|---|---|
| `employees` | All employee-related keys | Employee model write |
| `reports` | All report cache entries | Report generation complete |
| {add all found} | | |

### Conventions
- All TTLs use jitter: `$ttl + random_int(0, $ttl * 0.1)`
- Mutex pattern used for: {list high-stampede-risk keys}
- Cache prefix: `{app_name}_`

---

## 6. Queue Architecture

### Driver & Queues
```
QUEUE_CONNECTION=redis
```

| Queue name | Purpose | Workers | Priority |
|---|---|---|---|
| `critical` | Payments, auth events | 3 | Highest |
| `default` | General jobs | 5 | Normal |
| `emails` | Email sending | 2 | Normal |
| `reports` | Report generation | 1 | Low |
| {add all found} | | | |

### Job Registry
| Job class | Queue | Tries | Backoff | Timeout | has failed() | Idempotent |
|---|---|---|---|---|---|---|
| SendAttendanceReport | reports | 3 | [10,60,300] | 120s | ✅ | ✅ |
| SendWelcomeEmail | emails | 3 | [10,60,300] | 30s | ✅ | ✅ |
| {add all found} | | | | | | |

### Scheduled Commands
| Command | Schedule | Notes |
|---|---|---|
| `reports:generate-monthly` | `0 2 * * *` | Runs at 2am daily |
| `cache:warm-dashboard` | `*/5 * * * *` | Every 5 min |
| {add all found} | | |

---

## 7. Application Structure

### Architecture Pattern
{MVC | Service Layer | Repository Pattern | Action Pattern | CQRS}

### Route Summary
| File | Count | Auth required | Notes |
|---|---|---|---|
| routes/web.php | {N} | Partial | Guest + auth routes |
| routes/api.php | {N} | Yes (Sanctum) | All under /api/v1/* |
| {add others} | | | |

### Model Registry
| Model | Table | Key relationships | Observer | Caches invalidated |
|---|---|---|---|---|
| Employee | employees | belongsTo:Department, hasMany:Contract | EmployeeObserver | employee:{id}, dashboard:stats |
| Department | departments | hasMany:Employee | DepartmentObserver | dept:{id}:employees |
| {add all} | | | | |

### Service Layer (if present)
| Service | Responsibility | Used by |
|---|---|---|
| ReportService | Report generation | ReportController, GenerateReport job |
| {add all} | | |

---

## 8. Authentication & Authorization

- **Mechanism:** {Sanctum API tokens | Session-based | Passport OAuth}
- **Guards:** {web (session), api (sanctum)}
- **Roles/Permissions:** {Spatie Permission | Custom | None}
  - Roles defined: {admin, manager, employee, ...}
- **Multi-tenancy:** {Yes — tenant_id scoped | No}

---

## 9. HTTP Server Configuration

### Server: {Caddy | Nginx}

**Compression:** {✅ gzip + brotli enabled | ❌ Missing — add encode directive}
**Static asset cache:** {✅ max-age=31536000 on .css/.js/.woff2 | ❌ Missing}
**HTTPS:** {✅ Enforced | ❌ Not enforced}
**HTTP/2:** {✅ Enabled | ❌ Not enabled}
**Security headers:**
| Header | Status |
|---|---|
| Strict-Transport-Security | ✅ / ❌ |
| X-Content-Type-Options | ✅ / ❌ |
| X-Frame-Options | ✅ / ❌ |
| Content-Security-Policy | ✅ / ❌ |

---

## 10. Observability Stack

| Tool | Purpose | Status |
|---|---|---|
| Laravel Telescope | Request/query introspection | Dev/staging only |
| Sentry | Error tracking + performance | ✅ Configured |
| Prometheus + Grafana | Metrics dashboards | ✅ Self-hosted |
| Loki + Promtail | Log aggregation | ✅ Self-hosted |
| {add others} | | |

**Log channels:** {stack → [daily, stderr, sentry]}
**Error sample rate:** {SENTRY_TRACES_SAMPLE_RATE=0.1}

---

## 11. Deployment Pipeline

### Flow
```
Push to main
  → Jenkins pipeline
  → docker build
  → docker push (ECR)
  → ansible deploy
  → php artisan config:cache ✅ / ❌
  → php artisan route:cache  ✅ / ❌
  → php artisan view:cache   ✅ / ❌
  → php artisan migrate --force
  → composer dump-autoload --classmap-authoritative ✅ / ❌
  → health check
```

### OPcache
```ini
opcache.enable=1                    ✅ / ❌
opcache.memory_consumption=256      ✅ / ❌
opcache.validate_timestamps=0       ✅ / ❌ (must be 0 in production)
```

---

## 12. Environment Variables

### App
| Variable | Example | Notes |
|---|---|---|
| APP_NAME | EmployeeManager | — |
| APP_ENV | production | — |
| APP_KEY | base64:... | — |
| APP_URL | https://management.baust.edu.bd | — |
| APP_TIMEZONE | Asia/Dhaka | — |

### Database
| Variable | Example | Notes |
|---|---|---|
| DB_CONNECTION | pgsql | — |
| DB_HOST | postgres | — |
| DB_PORT | 5432 | — |
| DB_DATABASE | employee_db | — |
| {add all DB_* vars} | | |

### Cache & Queue
| Variable | Example | Notes |
|---|---|---|
| CACHE_DRIVER | redis | — |
| REDIS_HOST | redis | — |
| QUEUE_CONNECTION | redis | — |
| {add all} | | |

### Third-party Services
| Variable | Service | Notes |
|---|---|---|
| MAIL_MAILER | smtp | — |
| SENTRY_LARAVEL_DSN | https://... | Error tracking |
| {add all} | | |

---

## 13. Coding Conventions

### Cache
- Key format: `resource:id` or `resource:type:identifier`
  Examples: `employee:42`, `report:monthly:2025-03`, `dept:5:employees`
- TTL: always add jitter — `$baseTtl + random_int(0, (int)($baseTtl * 0.1))`
- Tags: group related keys — flush by tag on model write
- Never cache: authenticated user-specific pages at CDN level

### Database
- All FK columns must have an index
- Use `->withCount()` for counts, never `->relation->count()` in loops
- Use `->cursorPaginate()` for API list endpoints
- Always use `->select([...])` in API controllers — never `SELECT *`
- Soft deletes on: {list models}

### Jobs
- Always pass model ID, never model instance (use `SerializesModels`)
- Always define: `$tries`, `$backoff`, `$timeout`, `failed()`
- Jobs must be idempotent — safe to run more than once
- DLQ pattern: `failed()` method logs to `Log::critical` + dispatches
  to manual-review queue

### Controllers
- Thin controllers — business logic in Services
- Always return API Resources, never raw models
- Validation in Form Requests, never inline

### General
- No `env()` calls outside `config/` files
- No business logic in migrations
- Feature flags via config, not hardcoded `if` blocks

---

## 14. Off-Limits & Special Rules

| Scope | Rule | Reason |
|---|---|---|
| `attendance_records` table | No schema changes without DBA approval | Legacy system dependency |
| `/api/v1/*` routes | No response shape changes | External consumers (ADMS device) |
| `config/auth.php` | No changes without security review | — |
| {add others} | | |

---

## 15. Known Issues & Technical Debt

### Open Performance Issues
| ID | Description | Severity | File | Status |
|---|---|---|---|---|
| P-001 | Dashboard aggregate queries not cached | High | DashboardController.php:88 | Backlogged |
| {add from TODO/FIXME grep} | | | | |

### TODO / FIXME Registry
| Type | File | Line | Comment |
|---|---|---|---|
| TODO | app/Services/ReportService.php | 142 | "move to queue" |
| FIXME | app/Jobs/SendAttendanceReport.php | 67 | "retry logic broken" |
| {add all found} | | | |

### Deprecated Code Still In Use
| Method | File | Replacement |
|---|---|---|
| {list @deprecated methods} | | |

---

## 16. Open Questions

{List anything the agent could not confirm from the codebase.
These require manual verification by a human.}

- [ ] What is the production Redis version? Not found in docker-compose.yml
- [ ] Is PgBouncer used for connection pooling? Not found in config
- [ ] What is the actual production traffic (req/min)? Not in codebase
- [ ] Are there load balancer health check routes? Not found in routes/
- [ ] {add all unconfirmed items}

---

## 17. Agent Usage Notes

When an AI agent reads this file, it should:

1. **Never modify** files listed in Section 14 (Off-Limits)
2. **Always use** cache key patterns from Section 5 — do not invent new patterns
3. **Always apply** TTL jitter as defined in Section 13
4. **Verify** open questions (Section 16) before making assumptions
5. **Update** this file after major changes and increment the version at the top

---

*This file was generated by the context-generator agent.*
*Verify all [UNCONFIRMED] fields manually before relying on this document.*
*Keep this file updated — a stale CONTEXT.md is worse than no CONTEXT.md.*
```

---

## PHASE 3 — VALIDATION

Before writing the final file, run these self-checks:

1. Every table listed in Section 4 — does it correspond to an actual
   `Schema::create()` call in migrations? Remove any that don't.

2. Every cache key in Section 5 — does it appear in an actual
   `Cache::remember()` or `Cache::put()` call? Remove any that don't.

3. Every job in Section 6 — does it correspond to an actual class
   in `app/Jobs/`? Remove any that don't.

4. Every off-limits rule in Section 14 — is it grounded in something
   found in the codebase (a comment, a README note, a migration comment)?
   If not, move it to Open Questions.

5. Are there any fields marked `[UNCONFIRMED]`?
   Move all of them to Section 16 (Open Questions) with a specific
   question about what needs to be verified.

---

## PHASE 4 — OUTPUT

Write the completed `CONTEXT.md` to the repository root.

Then produce a short summary:

```
CONTEXT.md GENERATED
====================
Sections completed:      {N}/17
Fields confirmed:        {N}
Fields unconfirmed:      {N}  → see Section 16 (Open Questions)
Tables documented:       {N}
Cache keys documented:   {N}
Jobs documented:         {N}
Models documented:       {N}
Off-limits rules:        {N}
TODO/FIXMEs found:       {N}

Action required:
  → Verify {N} open questions in Section 16
  → Review off-limits rules in Section 14 for accuracy
  → Add production traffic baselines to Section 15 manually
    (agent cannot read production metrics)
```

---

## CONSTRAINTS

- Never read or write `.env` files (only `.env.example`)
- Never execute database queries to discover schema — read only migration files
- Never expose secrets, keys, or credentials in the output
- Do not invent facts — only document what is in the codebase
- CONTEXT.md must be valid Markdown, no broken tables
- If a section has no data (e.g., no jobs exist), write `None found.`
  rather than omitting the section

---

## INVOCATION

Read the codebase now. Begin with Phase 1 discovery, announce your
findings per section as you go, then generate CONTEXT.md in Phase 2,
validate in Phase 3, and write the file in Phase 4.

Start now.