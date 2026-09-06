# Software Architecture Document (SAD) — taskq-api

> Architecture for `taskq-api`: an ASGI HTTP service that exposes a REST API
> for task submission, query, and execution; persists state in a relational
> database; evolves schema via Alembic migrations; and enforces authentication,
> authorization, and rate limiting.

## 1. Architecture Overview

`taskq-api` is a single-process ASGI service built on FastAPI + SQLAlchemy 2.x +
Alembic. The system has five logical layers, matched one-to-one with source
directories so each layer becomes a CRG community with predictable cohesion:

1. **API layer** (`taskq_api/api/`) — HTTP handlers, request/response models,
   dependency wiring (auth, scope, rate-limit), RFC 7807 exception handlers,
   health/metrics endpoints.
2. **Service layer** (`taskq_api/service/`) — business logic: task CRUD,
   command runner, authentication, scope check, token-bucket rate limiter.
   This layer is where business rules live; handlers stay ≤ 40 lines per
   NFR-11.
3. **Repository layer** (`taskq_api/repository/`) — all data access through
   SQLAlchemy sessions; transaction boundary management; ORM queries only
   (no string-concatenated SQL, no raw `execute(text(...))` on user input).
4. **Models layer** (`taskq_api/models/`) — declarative ORM mapping for
   `tasks`, `api_keys`, `tags`, `task_tags`, `task_results`, `rate_buckets`.
   Pure data definitions; no business logic.
5. **Migrations layer** (`taskq_api/migrations/versions/`) — three Alembic
   revisions (v1, v2, v3) with reversible `upgrade`/`downgrade`, including a
   data-preserving v3 split of `tasks.result_json` into `task_results`.

Two **independence modules** (`config.py`, `errors.py`) are importable from
every layer without creating upward edges (per NFR-06).

The layering contract `api > service > repository > models` is enforced by
`import-linter` (NFR-06) and forbids `sqlalchemy` imports outside
`repository/` and `models/`.

### 1.1 System Verification Target

> **Every exit gate (2, 3 and 4)**: the harness executes `make verify-system`.
> A non-zero exit fails the gate.

**Makefile target**: `verify-system`
**Exercises**:
- `alembic upgrade head` against a real SQLite file DB (FR-07 v1→v2→v3)
- Full test suite (`pytest 03-development/tests -q`) with skipped == 0 (NFR-09)
- Service boot via `uvicorn taskq_api.app:app` + `/healthz`, `/readyz` smoke
- `alembic downgrade base` → `alembic upgrade head` round-trip verifying v3
  data preservation (FR-07 / §8 #12)
- `/v1/metrics` read so the metrics module executes in-process

The target deliberately invokes the delivered entry point — `uvicorn
taskq_api.app:app` — so the harness measures which high-risk modules
(`taskq_api.service.runner`, `taskq_api.service.auth`,
`taskq_api.repository.session`, `migrations.versions.v3_split_results`)
actually run end-to-end.

## 2. Module Design

### 2.1 Directory Structure

```
03-development/
├── src/
│   ├── alembic.ini
│   └── taskq_api/
│       ├── __init__.py
│       ├── app.py                       # FastAPI app factory + lifespan
│       ├── config.py                    # Pydantic settings (independence module)
│       ├── errors.py                    # RFC 7807 ProblemDetail (independence module)
│       ├── api/                         # CRG community: api
│       │   ├── __init__.py
│       │   ├── tasks.py                  # /v1/tasks CRUD router  [FR-01, FR-02]
│       │   ├── runs.py                   # /v1/tasks/{id}/runs    [FR-02]
│       │   ├── health.py                 # /healthz, /readyz      [FR-09]
│       │   ├── metrics.py                # /v1/metrics            [FR-09]
│       │   ├── deps.py                   # auth/scope/rate-limit dependencies [FR-03, FR-04, FR-05]
│       │   └── exception_handlers.py     # RFC 7807 mapping       [FR-10]
│       ├── service/                     # CRG community: service
│       │   ├── __init__.py
│       │   ├── task_service.py           # CRUD business logic    [FR-01]
│       │   ├── runner.py                 # async subprocess exec  [FR-02, FR-08]
│       │   ├── auth.py                   # API-key hashing + scope check [FR-03, FR-04]
│       │   ├── rate_limit.py             # token bucket algorithm [FR-05]
│       │   └── redaction.py              # sensitive-data scrub   [NFR-04]
│       ├── repository/                  # CRG community: repository
│       │   ├── __init__.py
│       │   ├── session.py                # Session factory + ctx manager [FR-06]
│       │   ├── task_repository.py        # task CRUD + cursor pagination [FR-01]
│       │   ├── api_key_repository.py     # API-key lookup         [FR-03]
│       │   ├── run_repository.py         # task_results writes    [FR-02]
│       │   └── rate_bucket_repository.py # row-locked bucket update [FR-05]
│       ├── models/                      # CRG community: models
│       │   ├── __init__.py
│       │   ├── base.py                   # Declarative base
│       │   ├── task.py                   # Task, Tag, TaskTag     [FR-01, FR-07]
│       │   ├── api_key.py                # ApiKey                 [FR-03, FR-07]
│       │   ├── task_result.py            # TaskResult             [FR-02, FR-07]
│       │   └── rate_bucket.py            # RateBucket             [FR-05, FR-07]
│       └── migrations/
│           ├── env.py
│           └── versions/
│               ├── v1_initial.py                  [FR-07]
│               ├── v2_tags_and_unique_name.py     [FR-07]
│               └── v3_split_results.py            [FR-07] (high-risk)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── .importlinter                        # layers contract (NFR-06)
├── .env.example                         # 12 TASKQ_* vars (§5.1)
├── requirements.txt
├── requirements.lock
├── requirements-dev.txt
└── Makefile                             # verify-system target
```

Six source communities: `api`, `service`, `repository`, `models`, `migrations`,
`scripts` (the last for utility scripts; not counted if empty). Each is a
CRG community; each contains a hub module imported by ≥ 70% of siblings.

### 2.2 FR → Module Traceability

| FR | Primary modules | Notes |
|----|-----------------|-------|
| FR-01 (Task CRUD) | `api.tasks`, `service.task_service`, `repository.task_repository`, `models.task` | cursor-based pagination; `selectinload` for `task_tags` |
| FR-02 (Run endpoint) | `api.tasks` (`POST /v1/tasks/{id}/run`), `api.runs`, `service.runner`, `repository.run_repository`, `models.task_result` | `asyncio.create_subprocess_exec` only |
| FR-03 (API Key auth) | `service.auth`, `repository.api_key_repository`, `api.deps`, `models.api_key` | SHA-256 + `hmac.compare_digest` |
| FR-04 (Scope authz) | `service.auth`, `api.deps` | single dependency; check before resource fetch |
| FR-05 (Rate limit) | `service.rate_limit`, `repository.rate_bucket_repository`, `models.rate_bucket` | row-level lock in single transaction |
| FR-06 (Persistence + tx) | `repository.session`, every `*_repository` | context-manager tx; service holds no `Session` |
| FR-07 (Migrations) | `migrations.versions.v1_initial`, `migrations.versions.v2_tags_and_unique_name`, `migrations.versions.v3_split_results` | v3 is high-risk; round-trip tested |
| FR-08 (Async runner) | `service.runner`, `app` (lifespan drain) | `asyncio.TaskGroup`; `process.kill()` + `await wait()` |
| FR-09 (Health/observability) | `api.health`, `api.metrics` | `/readyz` checks `alembic current == head` |
| FR-10 (RFC 7807) | `errors`, `api.exception_handlers`, every service module (raises `ProblemDetail`) | `correlation_id` in header + log |

### 2.3 Module Specifications

#### 2.3.1 `taskq_api.app`

| Attribute | Value |
|-----------|-------|
| Responsibility | FastAPI app factory; lifespan hooks for runner drain + readyz state |
| External Interface | `app: FastAPI` (ASGI), `python -m taskq_api` (CLI) |
| Dependencies | `taskq_api.config`, `taskq_api.api.*`, `taskq_api.service.runner` |

Logical constraints:
- Lifespan must `await runner.drain(timeout=settings.DRAIN_TIMEOUT)` on
  shutdown (FR-08).
- Registers CORS middleware with origins == `TASKQ_CORS_ORIGINS` (empty →
  reject all per NFR-02).

#### 2.3.2 `taskq_api.config`

| Attribute | Value |
|-----------|-------|
| Responsibility | Pydantic settings: 12 `TASKQ_*` env vars (SPEC §5.1) |
| External Interface | `get_settings() -> Settings` (lru_cache) |
| Dependencies | stdlib + pydantic only |

Logical constraints:
- Independence module per NFR-06 — importable from any layer.
- `TASKQ_DB_URL` value never appears in logs (NFR-04).

#### 2.3.3 `taskq_api.errors`

| Attribute | Value |
|-----------|-------|
| Responsibility | `ProblemDetail` dataclass + `ProblemException`; RFC 7807 body builder |
| External Interface | `raise ProblemException(status=..., type=..., detail=...)` |
| Dependencies | stdlib only |

Logical constraints:
- Independence module per NFR-06.
- `detail` is sanitized — never contains SQL, stack traces, file paths (FR-10).

#### 2.3.4 `taskq_api.api` (CRG community: `api`)

| Attribute | Value |
|-----------|-------|
| Responsibility | HTTP edge: routing, request validation, auth/scope/rate-limit deps, RFC 7807 mapping |
| External Interface | Routers mounted under `/v1` (and `/healthz`, `/readyz`) |
| Dependencies | `taskq_api.service.*`, `taskq_api.errors`, `taskq_api.config` |

Hub module: `api/deps.py` (shared by all routers — provides
`require_auth`, `require_scope`, `consume_rate_token`).

Logical constraints:
- Every handler ≤ 40 lines; business logic goes to `service/` (NFR-11).
- `/v1` routes uniformly use the same `require_auth` dependency (verified by
  test asserting one common dependency on each route — FR-04).
- `/healthz`, `/readyz` exempt from auth and rate limit (FR-03, FR-05).

#### 2.3.5 `taskq_api.service` (CRG community: `service`)

| Attribute | Value |
|-----------|-------|
| Responsibility | Business rules: CRUD orchestration, runner, auth, rate-limit |
| External Interface | Functions returning dataclasses / raising `ProblemException` |
| Dependencies | `taskq_api.repository.*`, `taskq_api.models.*`, `taskq_api.errors` |

Hub module: `service/auth.py` (imported by every other service for `APIContext`).

Logical constraints:
- No module in `service/` may import `sqlalchemy` (NFR-06 forbidden contract);
  it talks to the DB only via `repository/*`.
- `asyncio.CancelledError` is re-raised in every `except` block; never
  swallowed (NFR-03).
- `service/runner.py` is high-risk: timeout must `process.kill()` + `await
  process.wait()` to prevent orphans (FR-08, R8).

#### 2.3.6 `taskq_api.repository` (CRG community: `repository`)

| Attribute | Value |
|-----------|-------|
| Responsibility | Persistence + transaction boundary; ORM queries |
| External Interface | Functions returning ORM rows; `session_scope()` context manager |
| Dependencies | `sqlalchemy`, `taskq_api.models`, `taskq_api.config` |

Hub module: `repository/session.py` (every other repo imports
`session_scope` and `get_session`).

Logical constraints:
- Single context manager per request: commit on success, rollback on
  exception (FR-06, NFR-03).
- Rate-bucket updates use `SELECT … FOR UPDATE` (Postgres) / SQLite-equivalent
  `BEGIN IMMEDIATE` to prevent races (FR-05, R12).
- Pre-load relations with `selectinload` / `joinedload`; list endpoints must
  emit constant SQL count (NFR-01, R5).

#### 2.3.7 `taskq_api.models` (CRG community: `models`)

| Attribute | Value |
|-----------|-------|
| Responsibility | Declarative ORM mapping |
| External Interface | ORM classes imported by `repository/*` and Alembic |
| Dependencies | `sqlalchemy` |

Hub module: `models/base.py` (`Base = declarative_base()`).

Logical constraints:
- No business logic; only column definitions, relationships, indexes.
- `Task.name` carries unique index added in v2 migration.
- `TaskResult` is the v3 split target (no leftover `result_json` column).

#### 2.3.8 `taskq_api.migrations.versions`

| Attribute | Value |
|-----------|-------|
| Responsibility | Schema evolution across v1 → v2 → v3 |
| External Interface | `alembic upgrade head` / `downgrade base` |
| Dependencies | Alembic, `taskq_api.models.base` |

Logical constraints:
- Each revision has a working `downgrade()` (FR-07).
- `v3_split_results` is high-risk: must migrate `tasks.result_json` rows into
  `task_results`, then drop the column — reversible by inverse merge
  (R1, §8 #12).
- No `op.execute("DROP TABLE …")` shortcuts replacing real `downgrade()`
  (FR-07).

### 2.4 Cross-cutting Design

- **No circular dependencies** (architecture constraint).
- Every function body calls a sibling-hub function (CRG edge budget rule).
- Hub modules per community:
  `api/deps.py`, `service/auth.py`, `repository/session.py`,
  `models/base.py`, `migrations/env.py`.
- Source directories: 5 (`api`, `service`, `repository`, `models`,
  `migrations`) — within 3–6 CRG target.
- Community size cap ≤ 50 nodes; `repository/` has 5 files, well under cap.

## 3. Interfaces & Data Flows

### 3.1 External HTTP Surface

| Method | Path | Scope | Module | Returns |
|--------|------|-------|--------|---------|
| GET    | `/healthz` | — | `api.health` | `200 {"status":"ok"}` |
| GET    | `/readyz`  | — | `api.health` | `200` or `503 problem+json` |
| POST   | `/v1/tasks` | `write` | `api.tasks` | `201 TaskOut` |
| GET    | `/v1/tasks` | `read`  | `api.tasks` | `200 {items, next_cursor}` |
| GET    | `/v1/tasks/{id}` | `read`  | `api.tasks` | `200 TaskOut` / `404` |
| DELETE | `/v1/tasks/{id}` | `admin` | `api.tasks` | `204` / `404` |
| POST   | `/v1/tasks/{id}/run` | `write` | `api.tasks` | `202 {run_id}` |
| GET    | `/v1/tasks/{id}/runs` | `read`  | `api.runs`  | `200 [RunOut]` (new→old) |
| GET    | `/v1/metrics` | `admin` | `api.metrics` | `200 metrics payload` |

All `/v1/*` endpoints require `X-API-Key`; missing/invalid → `401`. Insufficient
scope → `403` (body must not leak resource existence). Rate limit exceeded →
`429` + `Retry-After`.

### 3.2 Request Data Flow (CRUD example: `POST /v1/tasks`)

```
client
  │  POST /v1/tasks {command, name}
  ▼
FastAPI router (api/tasks.py)
  │  ── depends_on ──► api/deps.require_auth("write")  ──► service/auth.py
  │                                                              │
  │                                                              ▼
  │                                                  repository/api_key_repository
  │                                                              │
  │                                                              ▼
  │                                                       models/api_key
  │  ◄── APIContext(key_id, scope) ──
  │
  │  ── pydantic TaskCreate validates body (FR-01) ──► 422 on failure
  ▼
service/task_service.create_task(ctx, payload)
  │  ── duplicate-name check via repository/task_repository ──► 409 on hit
  │  ── INSERT tasks + tags via repository/session ──► commit on success
  ▼
HTTP 201 {id, command, name, status:"pending", created_at}
```

Error paths route through `api/exception_handlers.py` → `ProblemDetail` body
with `correlation_id` echoed in `X-Correlation-Id` header.

### 3.3 Run Endpoint Flow (`POST /v1/tasks/{id}/run`)

```
client ─► api/tasks.run_task
          │  scope=write  (deps)
          ▼
service/runner.spawn(task, ctx)
          │  schedule on asyncio.TaskGroup (semaphore TASKQ_MAX_CONCURRENT)
          ▼
        return run_id  ──► 202 Accepted
        (background)
        asyncio.create_subprocess_exec(*shlex.split(command), shell=False)
          │  on exit/timeout:
          │    exit_code / stdout_tail / stderr_tail / duration_ms
          ▼
        repository/run_repository.insert(result)
          │  updates Task.status to done|failed|timeout
          ▼
        redaction.scrub(stdout/stderr) before persistence (NFR-04)
```

### 3.4 Migration Flow (FR-07 v3 round-trip)

```
alembic upgrade head
  v1: create tasks, api_keys
  v2: create tags, task_tags, idx tasks.name unique
  v3: create task_results;
      INSERT INTO task_results SELECT … FROM tasks WHERE result_json IS NOT NULL;
      -- application-managed backfill (deterministic order)
      ALTER TABLE tasks DROP COLUMN result_json;

alembic downgrade -1
  v3.rev: ALTER TABLE tasks ADD COLUMN result_json;
      UPDATE tasks SET result_json = json_object('exit_code', …) FROM task_results WHERE …;
      DROP TABLE task_results;
```

Round-trip verified by §8 #12: seed sample → upgrade → write samples →
downgrade -1 → upgrade → assert column-equality.

### 3.5 Component Diagram

```
┌──────────────────────── HTTP client ─────────────────────────┐
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS (uvicorn reverse-proxied)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  uvicorn  ─►  taskq_api.app  ─►  api/* (routers + deps)     │
│                                              │              │
│                                              ▼              │
│                                     service/* (business)    │
│                                              │              │
│                                              ▼              │
│                                   repository/* (ORM, tx)    │
│                                              │              │
│                                              ▼              │
│   ┌───────────────┐      models/*      ┌──────────────┐     │
│   │ migrations/   │◄──── schema ──────►│    SQLite    │     │
│   │  versions/    │                    │  (dev) /     │     │
│   │ v1,v2,v3      │                    │  Postgres    │     │
│   └───────────────┘                    │  (prod)      │     │
│                                        └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

## 4. NFR Handling

| NFR | Dimension | Architectural handling | Module / file |
|-----|-----------|------------------------|---------------|
| NFR-01 (perf + N+1) | performance | Cursor pagination on `?cursor=`; `selectinload`/`joinedload` for `task_tags`; SQLAlchemy event-listener test asserts constant SQL count for list endpoints; p95 budgets via `pytest-benchmark` | `repository/task_repository.py`, `models/task.py`, `tests/integration/test_perf.py` |
| NFR-02 (HTTP/DB security) | security | `shell=True` / `eval(` / `exec(` forbidden (grep gate 0 hit); ORM/parameterized queries only; SHA-256 + `hmac.compare_digest`; CORS default reject; `bandit -r` 0 HIGH/MEDIUM | `service/auth.py`, `service/runner.py`, `app.py`, `.importlinter` |
| NFR-03 (error/tx/async) | error_handling | `session_scope()` context manager enforces commit/rollback; no bare `except:` or `except Exception: pass`; `CancelledError` re-raised in every handler | `repository/session.py`, `service/*`, `api/exception_handlers.py` |
| NFR-04 (redaction) | security | `service/redaction.py` scrubs `(sk-…|token=…|Bearer …|postgres(ql)?://…)` lines to `[REDACTED]` before persistence, log emission, or metrics emission | `service/redaction.py`, `repository/run_repository.py`, `app.py` logging filter |
| NFR-05 (docs) | documentation | All public symbols carry docstring referencing `[FR-XX]` / `[NFR-XX]`; OpenAPI `summary`/`description` populated for every route | every module + `api/*` route decorators |
| NFR-06 (layering) | architecture_constraints | `api > service > repository > models` + `sqlalchemy` forbidden outside repository/models, both enforced by `import-linter` | `.importlinter` |
| NFR-07 (license) | license_compliance | `requirements.txt` (`==`) + `requirements.lock`; allowlist MIT/BSD-{2,3}-Clause/Apache-2.0/PSF; `pip-licenses --with-system`; SBOM at `08-config/SBOM.json` | `requirements*.txt`, `08-config/SBOM.json` |
| NFR-08 (mutation) | mutation_testing | `mutmut run` against `service/` + `repository/`; score ≥ 70; scope configured in `.methodology/harness_config.json` | `mutmut` config, `.methodology/harness_config.json` |
| NFR-09 (zero skip) | test_assertion_quality | No `pytest.skip`/`skipif`/`xfail`; no `--ignore`/`collect_ignore`; FR-07 migrations tested against a real SQLite file | `tests/**`, `tests/integration/test_migrations_roundtrip.py` |
| NFR-10 (integration coverage) | integration_coverage | `tests/integration/` ≥ 80% line coverage; httpx `AsyncClient(ASGITransport)`; covers every error code + drain + rate-limit | `tests/integration/` |
| NFR-11 (readability) | readability | MI ≥ 80, CC ≤ 10, file ≤ 400 LOC, dir ≤ 15 files, handler ≤ 40 LOC; `readability-v2` lint gate | `Makefile`, all modules |
| NFR-12 (verify-system) | execute_verification_target | `make verify-system` runs `alembic upgrade head` + tests + boot + `/healthz` `/readyz` smoke + `alembic downgrade base` → `upgrade head` round-trip; exits 0 + prints `verify-system: PASS` | `Makefile` |

> **Dimension vs type — NFR-03.** The `Dimension` column above names the *gate*
> dimension (`scoreable_dimension_names()` in
> `core/quality_gate/sab_parser.py`). The §5 SAB block `nfr_traceability.dimension`
> must textually match SRS.md's stated `**dimension**:` field; for NFR-03 both
> record `reliability` (`_NFR_TYPE_TO_DIM` maps `reliability → error_handling`
> for the gate scorer, so the §4 table shows the resolved gate dimension while
> §5 carries the SRS-stated dimension). For NFR-06..NFR-12 the `type:` token
> is the short form (`layering`, `licensing`, `mutation`, `testability`,
> `integration`, `maintainability`, `verifiability`) and the SRS `dimension:`
> already matches the gate dimension one-to-one — no split applies.

## 5. SAB Block (machine-readable — BINDING CONTRACT)

> **CONTRACT**: Field names, types, `sab:` root key, and `phase` as int must
> match `core/quality_gate/sab_parser.py:render_canonical_sab_template()`.
> Do NOT hand-write the YAML — paste from the canonical template and replace
> EXAMPLE values with your project's real values.
> Validate before committing: `python3 scripts/generate_sab.py --validate --project .`

<!-- SAB:START -->
```yaml
sab:
  version: "1.0"
  created_at: "2026-09-06"
  phase: 2  # MUST be int, NOT a string — parser raises on 'phase: "2"'
  project: "taskq-api"

  layers:
    - name: api
      modules:
        - "taskq_api.app"
        - "taskq_api.api.tasks"
        - "taskq_api.api.runs"
        - "taskq_api.api.health"
        - "taskq_api.api.metrics"
        - "taskq_api.api.deps"
        - "taskq_api.api.exception_handlers"
      allowed_dependencies: ["service"]
    - name: service
      modules:
        - "taskq_api.service.task_service"
        - "taskq_api.service.runner"
        - "taskq_api.service.auth"
        - "taskq_api.service.rate_limit"
        - "taskq_api.service.redaction"
      allowed_dependencies: ["repository", "models"]
    - name: repository
      modules:
        - "taskq_api.repository.session"
        - "taskq_api.repository.task_repository"
        - "taskq_api.repository.api_key_repository"
        - "taskq_api.repository.run_repository"
        - "taskq_api.repository.rate_bucket_repository"
      allowed_dependencies: ["models"]
    - name: models
      modules:
        - "taskq_api.models.base"
        - "taskq_api.models.task"
        - "taskq_api.models.api_key"
        - "taskq_api.models.task_result"
        - "taskq_api.models.rate_bucket"
      allowed_dependencies: []

  allowed_dependencies:
    - {from: api,        to: service}
    - {from: service,    to: repository}
    - {from: service,    to: models}
    - {from: repository, to: models}

  quality_targets:
    max_complexity: 10
    min_coverage: 100
    max_coupling: 0.3

  nfr_traceability:
    NFR-01: {dimension: performance,        type: performance,            target: "p95 GET task < 30ms; SQL count const", module: taskq_api.repository.task_repository}
    NFR-02: {dimension: security,           type: security,                target: "bandit 0 HIGH/MED; shell=True 0 hit", module: taskq_api.service.auth}
    NFR-03: {dimension: error_handling,     type: reliability,             target: "tx commit/rollback; CancelledError re-raised", module: taskq_api.repository.session}
    NFR-04: {dimension: security,           type: security,                target: "regex redaction on all sinks", module: taskq_api.service.redaction}
    NFR-05: {dimension: documentation,      type: documentation,          target: "100% public docstring with FR/NFR ref", module: taskq_api.api.tasks}
    NFR-06: {dimension: architecture_constraints, type: layering,        target: "import-linter exit 0; sqlalchemy forbidden outside repo/models", module: taskq_api.repository.session}
    NFR-07: {dimension: license_compliance,        type: licensing,       target: "pip-licenses allowlist only", module: requirements.lock}
    NFR-08: {dimension: mutation_testing,          type: mutation,        target: "mutation score >= 70 (service+repository)", module: taskq_api.service.runner, scope_layers: ["service", "repository"]}
    NFR-09: {dimension: test_assertion_quality,    type: testability,     target: "pytest skipped == 0", module: tests}
    NFR-10: {dimension: integration_coverage,      type: integration,     target: "tests/integration line >= 80%", module: tests.integration}
    NFR-11: {dimension: readability,               type: maintainability, target: "MI >= 80; CC <= 10; handler <= 40 LOC", module: taskq_api.api.tasks}
    NFR-12: {dimension: execute_verification_target, type: verifiability, target: "make verify-system exit 0 + PASS", module: Makefile}

  advisory_only: []

  gate_score_overrides: {}

  fr_module_traceability:
    FR-01: ["taskq_api.api.tasks", "taskq_api.service.task_service", "taskq_api.repository.task_repository", "taskq_api.models.task"]
    FR-02: ["taskq_api.api.tasks", "taskq_api.api.runs", "taskq_api.service.runner", "taskq_api.repository.run_repository", "taskq_api.models.task_result"]
    FR-03: ["taskq_api.service.auth", "taskq_api.repository.api_key_repository", "taskq_api.api.deps", "taskq_api.models.api_key"]
    FR-04: ["taskq_api.service.auth", "taskq_api.api.deps"]
    FR-05: ["taskq_api.service.rate_limit", "taskq_api.repository.rate_bucket_repository", "taskq_api.models.rate_bucket"]
    FR-06: ["taskq_api.repository.session", "taskq_api.repository.task_repository", "taskq_api.repository.api_key_repository", "taskq_api.repository.run_repository", "taskq_api.repository.rate_bucket_repository"]
    FR-07: ["taskq_api.migrations.versions.v1_initial", "taskq_api.migrations.versions.v2_tags_and_unique_name", "taskq_api.migrations.versions.v3_split_results"]
    FR-08: ["taskq_api.service.runner", "taskq_api.app"]
    FR-09: ["taskq_api.api.health", "taskq_api.api.metrics"]
    FR-10: ["taskq_api.errors", "taskq_api.api.exception_handlers"]

  architecture_constraints:
    - "no_circular_dependencies"
    - "api_>_service_>_repository_>_models"
    - "sqlalchemy_forbidden_outside_repository_and_models"

  high_risk_modules:
    - "taskq_api.service.runner"
    - "taskq_api.service.auth"
    - "taskq_api.repository.session"
    - "taskq_api.migrations.versions.v3_split_results"

  required_artifacts:
    - ".importlinter"
    - ".env.example"
    - "requirements.txt"
    - "requirements.lock"
    - "requirements-dev.txt"
    - "alembic.ini"
    - "Makefile"
    - ".methodology/harness_config.json"
```
<!-- SAB:END -->

Note: Fill in the YAML above — it is used for Drift Detection and gate scoring.
Generate: `python3 scripts/generate_sab.py --project . [--overwrite]`

---

## 6. Security Design (STRIDE-lite — machine-readable, BINDING CONTRACT)

> **CONTRACT**: Field names and the `security_design:` root key are parsed
> by `core/quality_gate/security_design.py:extract_security_block()`.
> Do NOT hand-write the YAML — paste from the canonical template and
> replace EXAMPLE values with your project's real values.
> Validate: `python3 harness_cli.py check-artifact-consistency --project .`

<!-- SEC:START -->
```yaml
security_design:
  version: "1.0"
  applicability: full
  justification: ""
  trust_boundaries:
    - id: TB-01
      name: "external HTTP input"
      description: "unauthenticated client requests crossing into the FastAPI router layer"
    - id: TB-02
      name: "API-key authenticated zone"
      description: "requests that passed X-API-Key + scope check, entering business service"
    - id: TB-03
      name: "DB transaction boundary"
      description: "repository layer writing to SQLite/Postgres inside a session_scope context manager"
    - id: TB-04
      name: "subprocess execution boundary"
      description: "service.runner spawning asyncio.create_subprocess_exec child processes"
    - id: TB-05
      name: "log + metrics sink"
      description: "stdout/stderr tails, error bodies, and /v1/metrics output reaching disk or wire"
  threats:
    - id: T-01
      boundary: TB-01
      category: tampering
      description: "malformed JSON body mutates task state without field validation"
      mitigation: "pydantic TaskCreate schema rejects unknown fields and constraint violations -> 422"
      owner_module: "taskq_api.api.tasks"
      nfr: NFR-02
      verified_by: "test_sec_t01_malformed_body_rejected"
    - id: T-02
      boundary: TB-01
      category: spoofing
      description: "forged or revoked X-API-Key bypasses authentication"
      mitigation: "SHA-256 lookup + hmac.compare_digest constant-time compare; revoked_at != NULL rejected"
      owner_module: "taskq_api.service.auth"
      nfr: NFR-02
      verified_by: "test_sec_t02_invalid_api_key_rejected"
    - id: T-03
      boundary: TB-02
      category: elevation_of_privilege
      description: "low-privilege token invokes admin endpoint (DELETE /v1/tasks/{id})"
      mitigation: "single require_scope dependency checks hierarchy (read<write<admin) BEFORE resource fetch, so 403 body never reveals existence"
      owner_module: "taskq_api.service.auth"
      nfr: NFR-02
      verified_by: "test_sec_t03_scope_enforced_for_delete"
    - id: T-04
      boundary: TB-03
      category: tampering
      description: "string-concatenated SQL mutates or extracts unintended rows"
      mitigation: "all queries via SQLAlchemy ORM or text(bind_params); import-linter blocks sqlalchemy import outside repository/models; grep gate forbids f-string/%/+ SQL"
      owner_module: "taskq_api.repository.task_repository"
      nfr: NFR-02
      verified_by: "test_sec_t04_no_string_sql"
    - id: T-05
      boundary: TB-04
      category: elevation_of_privilege
      description: "shell metacharacters in command field enable shell injection"
      mitigation: "asyncio.create_subprocess_exec(*shlex.split(command), shell=False) -- shell=True forbidden by grep gate; subprocess timeout enforced"
      owner_module: "taskq_api.service.runner"
      nfr: NFR-02
      verified_by: "test_sec_t05_shell_injection_blocked"
    - id: T-06
      boundary: TB-04
      category: denial_of_service
      description: "subprocess ignores timeout, leaks orphan processes"
      mitigation: "asyncio.wait_for + process.kill() then await process.wait(); TASKQ_MAX_CONCURRENT semaphore caps parallelism"
      owner_module: "taskq_api.service.runner"
      nfr: NFR-03
      verified_by: "test_sec_t06_timeout_kills_subprocess"
    - id: T-07
      boundary: TB-03
      category: denial_of_service
      description: "rate-limit bucket race allows infinite token consumption"
      mitigation: "single-transaction UPDATE with row-level lock (FOR UPDATE / BEGIN IMMEDIATE); over-limit returns 429 + Retry-After"
      owner_module: "taskq_api.repository.rate_bucket_repository"
      nfr: NFR-03
      verified_by: "test_sec_t07_rate_limit_atomic"
    - id: T-08
      boundary: TB-05
      category: information_disclosure
      description: "stdout/stderr tails leak secrets (sk-…, token=…, postgres://…) or DB URLs"
      mitigation: "service.redaction.scrub() replaces matching lines with [REDACTED] before persistence and metrics emission; correlation_id never carries payload"
      owner_module: "taskq_api.service.redaction"
      nfr: NFR-04
      verified_by: "test_sec_t08_secrets_redacted"
    - id: T-09
      boundary: TB-05
      category: information_disclosure
      description: "RFC 7807 problem+json detail field leaks stack traces, SQL fragments, or file paths"
      mitigation: "ProblemDetail.detail is a fixed whitelisted message; api/exception_handlers maps unhandled exceptions to generic detail; structured logs separately"
      owner_module: "taskq_api.api.exception_handlers"
      nfr: NFR-04
      verified_by: "test_sec_t09_problem_detail_no_internal_info"
    - id: T-10
      boundary: TB-01
      category: information_disclosure
      description: "CORS misconfiguration allows cross-origin credentialed access to /v1 endpoints"
      mitigation: "CORSMiddleware allow_origins derived from TASKQ_CORS_ORIGINS (empty -> reject all); credentials disabled"
      owner_module: "taskq_api.app"
      nfr: NFR-02
      verified_by: "test_sec_t10_cors_default_deny"
```
<!-- SEC:END -->

Note: `owner_module` names a module declared in the §5 SAB block;
`nfr` (optional) must exist in SRS.md; `verified_by` names the test that
proves the mitigation — from Phase 5 onward, `check-artifact-consistency`
blocks if that test doesn't exist yet. Threats also seed
`bug-hunt-targets`' adversarial-review targeting and force NFR-pattern
test cases in `derive_test_cases.md` Step 1c regardless of SRS keywords.
