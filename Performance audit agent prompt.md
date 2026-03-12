# Coding Agent Prompt: Full-Stack Performance & Caching Audit + Remediation

---

## ROLE & OBJECTIVE

You are a senior performance engineering agent. Your task is to perform a **comprehensive, non-destructive performance and caching audit** of this codebase, then **implement all safe improvements** directly. For changes that are risky, breaking, or require human decisions, you will produce a prioritized remediation plan instead of applying them blindly.

You must reason like an engineer, not a linter. Every change you make must be:
1. Justified by evidence found in the codebase (not assumptions)
2. Correct in the context of the existing architecture
3. Safe — no behavior changes to business logic, only performance improvements
4. Documented with a comment or migration note explaining what was changed and why

---

## PHASE 1 — CODEBASE DISCOVERY

Before touching a single file, build a complete picture of the system. Run the following discovery steps and record your findings.

### 1.1 Identify the Stack

Detect and record:
- Framework and version (`composer.json`, `package.json`)
- PHP version (`.php-version`, `Dockerfile`, `composer.json` platform requirements)
- Database drivers and versions (`config/database.php`, `docker-compose.yml`, `.env.example`)
- Cache driver in use (`config/cache.php`, `.env` — file, redis, memcached, array, dynamodb)
- Queue driver in use (`config/queue.php`)
- Session driver (`config/session.php`)
- HTTP server (Nginx, Caddy, Apache — check `Caddyfile`, `nginx.conf`, Dockerfile)
- CDN or reverse proxy presence (Cloudflare, Nginx upstream headers)
- Any existing performance packages (`laravel/octane`, `laravel/horizon`, `spatie/laravel-query-builder`, `barryvdh/laravel-debugbar`, etc.)

### 1.2 Map the Application Surface

- List all route files (`routes/web.php`, `routes/api.php`, etc.) and count the number of routes
- Identify the heaviest routes: any route with complex middleware stacks, nested resource controllers, or deeply nested relationships
- List all Eloquent models and their relationships (`hasMany`, `belongsToMany`, `hasManyThrough`)
- List all database tables and their approximate row counts if accessible via seeders, factories, or migration comments
- Identify all scheduled commands (`app/Console/Kernel.php`)
- Identify all queue jobs (`app/Jobs/`)
- Identify all event listeners (`app/Listeners/`, `EventServiceProvider`)
- List all service providers with `boot()` methods that do work (`app/Providers/`)

### 1.3 Identify Existing Caching

Grep for all cache usage in the codebase:
```
Cache::get, Cache::put, Cache::remember, Cache::forget, Cache::tags
Redis::get, Redis::set, Redis::setex
config('cache'), cache()
```
Record: what keys are cached, what TTLs are used, whether tags are used, whether invalidation exists for each cached item.

### 1.4 Identify Existing Query Optimization

Grep for:
```
->with(, ->withCount(, ->load(, ->loadCount(  ← eager loading
->select(                                      ← field selection
->chunk(, ->cursor(                            ← large dataset handling
->index, Schema::table                         ← migrations with indexes
DB::raw(, selectRaw(                           ← raw queries
->paginate(, ->simplePaginate(, ->cursorPaginate(  ← pagination
```

---

## PHASE 2 — AUTOMATED DETECTION

Run each of the following analyses in sequence. For each issue found, record: **file path**, **line number**, **issue type**, **severity** (Critical / High / Medium / Low), and **recommended fix**.

---

### 2.1 N+1 Query Detection

Search for every `foreach`, `for`, or `->each()` loop that contains an Eloquent relationship access (`->relation`, `->relationName`) where the relationship was not eager-loaded in the originating query.

**Pattern to find:**
```php
// PROBLEM: relationship accessed inside loop without eager load
$employees = Employee::all();           // ← no ->with()
foreach ($employees as $emp) {
    echo $emp->department->name;        // ← triggers query per iteration
    echo $emp->contracts->count();      // ← another query per iteration
}
```

**Fix to apply:**
```php
$employees = Employee::with(['department', 'contracts'])->get();
```

**For count-only access**, use `withCount` instead of loading the full relationship:
```php
// Instead of: $emp->contracts->count()
Employee::withCount('contracts')->get();
// Access as: $emp->contracts_count
```

**After fixing**, add this to `AppServiceProvider::boot()` if not already present:
```php
\Illuminate\Database\Eloquent\Model::preventLazyLoading(
    ! app()->isProduction()
);
```
This will throw an exception in development the moment a new N+1 is introduced.

---

### 2.2 Missing Database Indexes

For every migration file, extract all columns used in foreign keys (`->foreign()`, `->foreignId()`). Check whether a corresponding `->index()` or `->primary()` exists for that column.

**Automatically add missing FK indexes:**
```php
// Generate a new migration:
// database/migrations/YYYY_MM_DD_add_missing_fk_indexes.php
Schema::table('employees', function (Blueprint $table) {
    $table->index('department_id');   // if missing
    $table->index('created_at');      // if used in ORDER BY / WHERE
});
```

Also grep routes and controllers for `->orderBy(`, `->where(`, `->groupBy(` column names, then cross-reference against migration index definitions. Flag any column used in these clauses that lacks an index.

**Generate a single migration file** with all missing indexes grouped by table.

---

### 2.3 Unbounded Queries

Search for any query that:
- Uses `->get()` without `->limit()`, `->take()`, or `->paginate()` on a model that could have unbounded rows
- Uses `->all()` on models that represent high-volume tables (orders, logs, events, audit trails, notifications — detect by model name and table name heuristics)
- Passes unvalidated user input directly to `->skip()` / `->offset()` (OFFSET pagination problem)

**Fix pattern for OFFSET pagination:**
```php
// BEFORE: slow at large offsets
$records = Record::orderBy('id')->skip($request->page * 50)->take(50)->get();

// AFTER: cursor pagination (constant time regardless of page depth)
$records = Record::orderBy('id')
    ->when($request->cursor, fn($q) => $q->where('id', '>', $request->cursor))
    ->take(50)
    ->get();
```

---

### 2.4 SELECT * Usage

Find all queries using `->get()` or `DB::table()->get()` without a preceding `->select(...)` where the result is used for display/API response only (not for Eloquent operations that require the full model).

Flag these for review. Where the controller immediately maps to an API resource or array, add explicit `->select([...])`.

---

### 2.5 Missing Cache on Expensive Operations

Search for:
1. Any controller method or service method that queries an aggregate (`->count()`, `->sum()`, `->avg()`, `->max()`) without a surrounding `Cache::remember()`
2. Any scheduled command that generates a report and stores to DB — check if the result is also put in cache for fast dashboard access
3. Any `config()` call inside a loop (should be extracted before the loop)
4. Any `env()` call outside of a config file (should never be used directly in controllers/services)

**Fix for aggregate caching:**
```php
// BEFORE
$stats = [
    'total' => Employee::count(),
    'active' => Employee::where('status', 'active')->count(),
    'departments' => Department::withCount('employees')->get(),
];

// AFTER
$stats = Cache::remember('dashboard:employee_stats', 300, fn() => [
    'total' => Employee::count(),
    'active' => Employee::where('status', 'active')->count(),
    'departments' => Department::withCount('employees')->get(),
]);
```

---

### 2.6 Cache Invalidation Audit

For every `Cache::remember()` or `Cache::put()` found in Phase 1.3, check:

1. Is there a corresponding `Cache::forget()` or `Cache::tags(...)->flush()` somewhere in the codebase?
2. Is the key invalidated in the relevant model's observer, event listener, or controller after a write (create/update/delete)?
3. Are tag-based groups used where multiple related cache keys need coordinated invalidation?

**Flag** any cached item with no invalidation path. Generate the missing invalidation code:

```php
// If Employee model is updated, bust related caches
// app/Observers/EmployeeObserver.php
class EmployeeObserver
{
    public function saved(Employee $employee): void
    {
        Cache::forget("employee:{$employee->id}");
        Cache::forget('dashboard:employee_stats');
        Cache::tags(['employees'])->flush();
    }

    public function deleted(Employee $employee): void
    {
        Cache::forget("employee:{$employee->id}");
        Cache::forget('dashboard:employee_stats');
    }
}

// Register in AppServiceProvider::boot():
Employee::observe(EmployeeObserver::class);
```

---

### 2.7 Cache Stampede Risk Detection

For every `Cache::remember()` found, check:

1. Is the TTL a fixed round number (3600, 86400, etc.)? → Flag for jitter
2. Is the key high-traffic (used in a controller on a frequently accessed route)? → Flag for mutex or stale-while-revalidate
3. Is the computation expensive (contains DB aggregate, external HTTP call, or report generation)? → Flag for pre-warming

**Apply TTL jitter to all flagged keys:**
```php
// BEFORE
Cache::remember('report:monthly', 3600, fn() => Report::generate());

// AFTER
Cache::remember('report:monthly', 3600 + random_int(0, 300), fn() => Report::generate());
```

**Apply mutex lock to high-traffic + expensive keys:**
```php
function getCachedReport(): mixed
{
    $value = Cache::get('report:monthly');
    if ($value !== null) return $value;

    $lock = Cache::lock('lock:report:monthly', 30);
    
    return $lock->block(10, function () {
        // Check again after acquiring lock (another process may have populated it)
        return Cache::remember('report:monthly', 3600 + random_int(0, 300), fn() => Report::generate());
    });
}
```

---

### 2.8 Queue Job Analysis

For every job class in `app/Jobs/`:

1. **Idempotency check:** Does the job have a unique mechanism (idempotency key, `upsert`, `updateOrCreate`, or `firstOrCreate`) that makes it safe to run more than once? If not, flag it.

2. **Payload size check:** Does the job serialize full Eloquent model instances? If yes, replace with `SerializesModels` trait and model ID passing.

3. **Missing `failed()` method:** Does the job have a `failed(Throwable $e)` method for DLQ handling? If not, generate one.

4. **Retry configuration:** Does the job define `$tries` and `$backoff`? If not, add sensible defaults.

**Standard job hardening template to apply:**
```php
class ProcessReport extends Job
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public array $backoff = [10, 60, 300];  // exponential-ish backoff
    public int $timeout = 120;
    public bool $failOnTimeout = true;

    // Pass ID, not model — model fetched fresh inside handle()
    public function __construct(private readonly int $reportId) {}

    public function handle(): void
    {
        $report = Report::findOrFail($this->reportId);
        // ... logic
    }

    public function failed(\Throwable $e): void
    {
        Log::critical('ProcessReport job failed permanently', [
            'report_id' => $this->reportId,
            'error'     => $e->getMessage(),
            'trace'     => $e->getTraceAsString(),
        ]);
        // Optionally: notify Slack, dispatch to manual-review queue
    }
}
```

---

### 2.9 Session & Cookie Driver Audit

Check `config/session.php`:

- If driver is `file` and the application has more than one server instance: **must change to `redis`** (file sessions don't work across multiple servers)
- If driver is `database`: flag as higher latency than Redis for session reads
- If `SESSION_LIFETIME` is very long (> 1440 minutes): flag as security risk

---

### 2.10 Configuration & Bootstrap Performance

1. Check if `php artisan config:cache` and `php artisan route:cache` are run in the deployment pipeline (check `Dockerfile`, CI scripts, `deploy.sh`, Makefile). If not, add them.

2. Check `composer.json` for `"optimize-autoloader": true` in the `config` section. Add if missing.

3. Check for `env()` calls outside of `config/` files — these break config caching. Grep for `env(` in `app/`, `routes/`, `database/`. Move each to a config file.

```php
// WRONG: breaks config:cache
class SomeService {
    public function __construct() {
        $this->key = env('SOME_KEY'); // ← breaks when config cached
    }
}

// CORRECT
// config/services.php:
'some_key' => env('SOME_KEY'),

// SomeService.php:
$this->key = config('services.some_key');
```

---

### 2.11 HTTP Response Optimization

Check the HTTP server config file (Caddyfile, nginx.conf):

**Caddy — verify these directives exist:**
```caddyfile
example.com {
    encode zstd gzip          # compression — add if missing

    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        Cache-Control "no-store" # default; override per-path
        -Server                  # hide Caddy version
    }

    # Static assets: long cache (assume hashed filenames from Vite)
    @static path_regexp \.(css|js|woff2?|png|jpg|webp|avif|svg|ico)$
    header @static Cache-Control "public, max-age=31536000, immutable"

    # API routes: no cache by default
    @api path /api/*
    header @api Cache-Control "no-store"
}
```

**Nginx — verify these directives exist:**
```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml;
gzip_min_length 1000;

location ~* \.(css|js|woff2?|png|jpg|webp|avif)$ {
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

---

### 2.12 Laravel OPcache Verification

Check Dockerfile or PHP config for OPcache settings. If missing, add:

```ini
; php.ini / docker/php/opcache.ini
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0    ; 0 = never check for file changes (production only)
opcache.save_comments=1
opcache.fast_shutdown=1
```

For development: `opcache.validate_timestamps=1` so file changes are picked up.

---

### 2.13 Database Connection Pool Audit

Check `config/database.php` for connection pool settings:

```php
'mysql' => [
    'options' => [
        PDO::ATTR_PERSISTENT => false,  // persistent connections — usually leave false with Laravel
    ],
    // These should be set:
    'connect_timeout' => 5,
    'read_timeout'    => 60,
],
```

Check `docker-compose.yml` or environment for PgBouncer / ProxySQL presence. If the application uses PostgreSQL and runs more than 2 replicas, flag PgBouncer as a recommended addition.

---

### 2.14 Frontend Asset Audit

Check `vite.config.js` or `webpack.mix.js`:

1. Is code splitting configured? (Dynamic imports for route-level splitting)
2. Is the bundle analyzed? (Add `rollup-plugin-visualizer` for Vite if not present)
3. Are images processed through an optimizer pipeline?
4. Is `laravel-vite-plugin` configured with correct manifest-based cache busting?

Check `resources/views/` for:
- `<img>` tags missing `loading="lazy"` on below-the-fold images
- `<img>` tags missing `width` and `height` attributes (causes CLS)
- `<script>` tags without `defer` or `async` in `<head>`
- External font imports (`fonts.googleapis.com`) — flag for self-hosting

---

## PHASE 3 — IMPLEMENT CHANGES

Apply the following changes in order of safety. Stop and report before applying anything in the "Requires Human Decision" category.

### 3.1 Safe to Apply Automatically

These changes are non-breaking and should be applied directly:

| # | Change | Files Affected |
|---|---|---|
| 1 | Add `preventLazyLoading()` to AppServiceProvider | `app/Providers/AppServiceProvider.php` |
| 2 | Add `withCount()` in place of lazy count access | Controller / Service files |
| 3 | Add explicit `->with([...])` for detected N+1 relationships | Controller / Service files |
| 4 | Add `->select([...])` to queries in API resources | Controller files |
| 5 | Add TTL jitter to all `Cache::remember()` calls | All Cache usage files |
| 6 | Add `failed()` method to all Job classes missing it | `app/Jobs/**` |
| 7 | Add `$tries`, `$backoff`, `$timeout` to Job classes | `app/Jobs/**` |
| 8 | Replace `SerializesModels` + pass ID instead of model | `app/Jobs/**` |
| 9 | Move `env()` calls from app code into config files | `app/**`, `config/**` |
| 10 | Add `config:cache`, `route:cache`, `view:cache` to deploy script | `Dockerfile` / deploy scripts |
| 11 | Add OPcache ini settings | `docker/php/opcache.ini` or `php.ini` |
| 12 | Add compression directives to Caddyfile/nginx.conf | Server config |
| 13 | Add static asset cache headers to server config | Server config |
| 14 | Add `loading="lazy"` to below-fold `<img>` tags | Blade templates |
| 15 | Add `width` and `height` to `<img>` tags | Blade templates |
| 16 | Add `defer` to `<script>` tags in `<head>` | Blade layout files |
| 17 | Generate migration for missing FK indexes | New migration file |
| 18 | Add Model Observers for cache invalidation where missing | `app/Observers/**` |

---

### 3.2 Requires Human Decision (Produce Plan Only)

For the following, do **not** change code. Instead, produce a **numbered remediation plan** with effort estimate, risk rating, and exact code diff to apply:

| # | Change | Reason for Human Review |
|---|---|---|
| 1 | Add Redis caching to identified expensive queries | Requires TTL decision + cache key naming convention agreement |
| 2 | Add Cache mutex to stampede-risk keys | Requires decision on wait-or-stale behavior |
| 3 | Switch session driver to `redis` | Requires environment change + session invalidation |
| 4 | Add cursor pagination to OFFSET pagination routes | May change API contract (different `page` param format) |
| 5 | Add read replica config | Requires infrastructure provisioning |
| 6 | Replace heavy full-table queries with paginated/chunked versions | May affect existing API consumers |
| 7 | Add job rate limiting / throttle middleware | Requires understanding of SLA and throughput targets |
| 8 | Introduce PgBouncer | Infrastructure change |
| 9 | Add full-text search index (FULLTEXT / tsvector) | Schema migration on large table — needs downtime planning |
| 10 | Implement cache pre-warming scheduled commands | Requires defining which keys to pre-warm and when |

---

## PHASE 4 — OUTPUT FORMAT

After completing all phases, produce the following output:

### 4.1 Executive Summary

```
PERFORMANCE AUDIT REPORT
========================
Files analyzed:         [N]
Issues found:           [N]
  Critical:             [N]   (data correctness risk or severe performance degradation)
  High:                 [N]   (significant impact, safe to fix)
  Medium:               [N]   (moderate impact)
  Low:                  [N]   (minor improvements)

Changes applied:        [N]
Changes recommended:    [N]   (require human decision)

Estimated performance impact of applied changes:
  N+1 queries eliminated:         [N queries removed per request on affected endpoints]
  Cache invalidation gaps fixed:  [N]
  Missing indexes added:          [N]
  Job hardening applied:          [N jobs]
```

### 4.2 Issue Registry

For every issue found, produce one row:

```
| ID  | Severity | File | Line | Issue | Fix Applied | Notes |
|-----|----------|------|------|-------|-------------|-------|
| P01 | Critical | app/Http/Controllers/EmployeeController.php | 42 | N+1: department relationship loaded in loop (18 queries per request) | ✅ Applied | Added ->with('department') |
| P02 | High     | app/Jobs/SendAttendanceReport.php | 15 | No failed() method — silent job loss on failure | ✅ Applied | Added failed() with Log::critical |
| P03 | High     | database/migrations/2024_01_create_attendance_table.php | 22 | Missing index on user_id FK | ✅ Applied | Generated migration |
| P04 | Medium   | app/Http/Controllers/DashboardController.php | 88 | COUNT query with no cache on high-traffic route | ⚠️ Plan Only | See Remediation Plan #1 |
```

### 4.3 Remediation Plan (for Human-Decision items)

For each deferred item, produce:

```markdown
## PLAN #1 — Cache Dashboard Statistics

**Issue:** DashboardController::index() runs 4 aggregate queries on every page load.
           At 100 req/min, this is 400 unnecessary DB queries/minute.

**Affected file:** app/Http/Controllers/DashboardController.php, lines 85–103

**Risk:** Low. Stats are display-only. 5-minute staleness is acceptable.

**Effort:** 30 minutes

**Recommended TTL:** 300 seconds (5 minutes) with ±60s jitter

**Exact diff to apply:**
--- a/app/Http/Controllers/DashboardController.php
+++ b/app/Http/Controllers/DashboardController.php
@@ -85,10 +85,14 @@
-    $stats = [
-        'total_employees' => Employee::count(),
-        'active'          => Employee::where('status', 'active')->count(),
-        'on_leave'        => Leave::activeToday()->count(),
-        'departments'     => Department::withCount('employees')->get(),
-    ];
+    $stats = Cache::remember('dashboard:stats', 300 + random_int(0, 60), fn() => [
+        'total_employees' => Employee::count(),
+        'active'          => Employee::where('status', 'active')->count(),
+        'on_leave'        => Leave::activeToday()->count(),
+        'departments'     => Department::withCount('employees')->get(),
+    ]);

**Invalidation required:** Add to EmployeeObserver::saved() and LeaveObserver::saved():
    Cache::forget('dashboard:stats');
```

### 4.4 Files Changed

List every file modified with a one-line description of what changed.

### 4.5 New Files Created

List every new file created (migrations, observers, config snippets) with path and purpose.

---

## CONSTRAINTS & GUARDRAILS

You must not:
- Change any business logic, validation rules, or data transformation
- Rename models, methods, routes, or database columns
- Remove any existing functionality
- Modify `.env` files directly (only `.env.example` if adding new variables)
- Run `php artisan migrate` — only generate migration files
- Delete any existing cache keys without first confirming their invalidation path

You must:
- Run `php artisan test` (or the project's test command) after each phase if tests exist
- Check if a change would break existing tests before applying
- Add a `// PERF:` comment above every change you make, explaining the issue it solves
- Commit-message-quality explanation for every file changed

---

## INVOCATION

Begin with Phase 1. Output your discovery findings before proceeding to Phase 2. Ask for confirmation before proceeding to Phase 3 if any Critical-severity issues were found that require human decisions.

Start now.