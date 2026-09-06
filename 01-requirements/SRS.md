# Software Requirements Specification (SRS) — taskq-api

> Source of truth: project-root `SPEC.md` v1.0.0 (the canonical spec).
> This SRS transcribes 100% of `### FR-01..FR-10` and `### NFR-01..NFR-12`
> headings from the canonical spec, with no invention, no silent omission
> of TBD/TODO/placeholders, and no prescriptive clauses beyond the
> canonical phrasing. Every acceptance criterion carries a stable
> `AC-<n>.<m>` (FR) or `AC-N<n>.<m>` (NFR) identifier for downstream
> `TEST_SPEC.md` citation and `check_ac_test_spec_coverage` counting.

## 1. Introduction

- **Project name**: `taskq-api`
- **Purpose**: HTTP-service a task queue — submit, query, and run tasks via a REST API; persist data in a relational database; evolve schema via real migrations; provide authentication, authorization and rate-limiting.
- **Language / runtime**: Python 3.11
- **Shape**: ASGI service launched as `uvicorn taskq_api.app:app`; a `python -m taskq_api` management entrypoint is provided (for `migrate` / `seed` / `healthcheck`).
- **Validation round**: harness-methodology progressive-validation testbed Round 2 (Python backend + database). Round 1 (`taskq-plus`) lit the `license_compliance`, `architecture_constraints`, `mutation_testing`, `test_assertion_quality` dimensions but was a single-process CLI.
- **Round 2 specific intents** (verbatim from canonical SPEC.md §0 "本輪設計意圖"):
  - HTTP layer → `security` scanner reaches `authn/authz/input boundaries`, not only subprocess
  - Real database → ORM, transactions, connection pool, N+1 are all first-class
  - Real schema migration → Alembic three-step evolution with data migration and reversible `downgrade`
  - Async path → `async def` endpoints + `asyncio.TaskGroup` background runner

## 2. Constraints

Constraints transcribed verbatim from canonical SPEC.md §1 / §2 / §5.3 / §7:

- **C-1** (verbatim from SPEC.md §1): Project language is Python 3.11; shape is an ASGI service launched by `uvicorn taskq_api.app:app`, with a separate `python -m taskq_api` management entrypoint.
- **C-2** (verbatim from SPEC.md §2 tech-stack table): HTTP framework is FastAPI (ASGI); data validation uses `pydantic` v2 request/response models; ORM is SQLAlchemy 2.x (declarative + explicit `Session` transaction boundaries); database is SQLite for dev/test, PostgreSQL for production (same ORM model); migrations use Alembic (v1 → v2 → v3 with `downgrade` for each); async uses `async def` endpoints + `asyncio.TaskGroup` background runner; auth uses `X-API-Key` header with hashed comparison (no plaintext storage); authz uses per-token scope `read` / `write` / `admin`; rate limiting uses a per-token token bucket; error contract is RFC 7807 `application/problem+json`; task execution uses `asyncio.create_subprocess_exec` with `shell=True` forbidden; layering constraint is enforced via `import-linter`.
- **C-3** (verbatim from SPEC.md §5.3 "專案側必備設定檔"): `.importlinter`, `requirements.txt` + `requirements.lock`, `requirements-dev.txt`, `alembic.ini` + `migrations/versions/`, `.env.example`, `.methodology/harness_config.json`, `Makefile` are all required and non-optional.
- **C-4** (verbatim from SPEC.md §7 error table): All non-2xx responses are `application/problem+json`. `asyncio.CancelledError` is **not** in the table — it must propagate upward, never convert to HTTP 500.
- **C-5** (verbatim from SPEC.md §10 "CRG 校準鐵律"): `crg_cohesion_healthy` MUST stay at the default value; it MUST NOT be downgraded to pass the project.
- **C-6** (verbatim from SPEC.md §10 "高風險模組"): `taskq_api.service.runner`, `taskq_api.service.auth`, `taskq_api.repository.session`, `migrations/versions/v3_split_results.py` are high-risk modules and require per-module TDD coverage.
- **C-7** (verbatim from SPEC.md §11 monitoring table): All metrics listed in §11 are Quality-Gate-aligned thresholds; failure on any of them blocks gate advancement.

## 3. Functional Requirements (FRs)

Each FR below transcribes one `### FR-XX` heading from canonical SPEC.md verbatim. All ACs carry stable identifiers of the form `AC-<n>.<m>`.

> DERIVED: SPEC.md §3 FR-01 (lines 79–92) + §8 #4-#8 — English section-summary prose is a derivative paraphrase of the Chinese source; the bullet ACs below quote canonical phrases (e.g. "驗證規則同第 1 輪 FR-01(非空 / ≤1000 字元 / 注入字元黑名單 / 名稱唯一);違反 → **HTTP 422** + problem+json") verbatim where possible; `implementation_functions` in the machine-readable block are forward-looking hints with no canonical source.

### FR-01: 任務資源 CRUD API

Canonical source: SPEC.md §3 FR-01 (lines 79–92).

- `POST /v1/tasks` (scope `write`) — create a task; body validated by the `TaskCreate` pydantic model.
- `GET /v1/tasks/{id}` (scope `read`) — fetch a single task's full fields.
- `GET /v1/tasks` (scope `read`) — paginated list supporting `?status=`, `?limit=`, `?cursor=`.
- `DELETE /v1/tasks/{id}` (scope `admin`) — delete the task (cascading result rows in the same transaction).

**Validation rules** (verbatim from canonical): "驗證規則同第 1 輪 FR-01(非空 / ≤1000 字元 / 注入字元黑名單 / 名稱唯一);違反 → **HTTP 422** + problem+json".

**Acceptance criteria**:

- **AC-1.1**: A `POST /v1/tasks` request with a valid body and `write`-scope key returns **HTTP 201** with a task id in the body. Verified by `tests/.../test_fr01_post_returns_201`. Per SPEC.md §8 #4.
- **AC-1.2**: A `POST /v1/tasks` request missing `X-API-Key` returns **HTTP 401** + problem+json. Verified by `tests/.../test_fr01_post_no_api_key_returns_401`. Per SPEC.md §8 #5.
- **AC-1.3**: A `POST /v1/tasks` request with an invalid body (empty name, name > 1000 chars, blacklisted injection chars, or duplicate name) returns **HTTP 422** + problem+json (or **HTTP 409** for duplicate name per FR-01's rule "名稱唯一" + §8 #8). Verified by `tests/.../test_fr01_post_invalid_body_returns_422_or_409`. Per SPEC.md §3 FR-01 and §8 #8.
- **AC-1.4**: A `POST /v1/tasks` request with a `name` that already exists returns **HTTP 409** + problem+json (`type=/errors/conflict`). Verified by `tests/.../test_fr01_post_duplicate_name_returns_409`. Per SPEC.md §8 #8.
- **AC-1.5**: A `GET /v1/tasks/{unknown_id}` returns **HTTP 404** + problem+json (`type=/errors/not-found`). Verified by `tests/.../test_fr01_get_unknown_returns_404`. Per SPEC.md §8 #7.
- **AC-1.6**: The `GET /v1/tasks` list endpoint uses **cursor-based pagination** (not offset) and returns a default `limit` of 50 with a hard ceiling of 200. A request with `limit > 200` returns **HTTP 422**. Verified by `tests/.../test_fr01_list_uses_cursor_pagination` and `test_fr01_list_limit_above_200_returns_422`. Per SPEC.md §3 FR-01.
- **AC-1.7**: A `DELETE /v1/tasks/{id}` with a `write` key (non-admin) returns **HTTP 403** and the body does NOT disclose whether the resource exists. Verified by `tests/.../test_fr01_delete_non_admin_returns_403_no_existence_leak`. Per SPEC.md §8 #6.
- **AC-1.8**: `DELETE /v1/tasks/{id}` with an `admin` key removes the task and its result rows in the same transaction (atomic). Verified by `tests/.../test_fr01_delete_admin_atomic`. Per SPEC.md §3 FR-01 ("連同結果列,同一交易").

> DERIVED: SPEC.md §3 FR-02 (lines 93–100) — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### FR-02: 任務執行端點

Canonical source: SPEC.md §3 FR-02 (lines 93–100).

- `POST /v1/tasks/{id}/run` (scope `write`) returns **HTTP 202 Accepted** with `run_id`.
- Execution uses `asyncio.create_subprocess_exec(*shlex.split(command))`; **`shell=True` is forbidden**; per-task timeout is `TASKQ_TASK_TIMEOUT`.
- State machine: `pending → running → done | failed | timeout`.
- Results are written to the `task_results` table (FR-07 v3 schema) with columns `exit_code` / `stdout_tail` / `stderr_tail` / `duration_ms` / `finished_at`.
- `GET /v1/tasks/{id}/runs` (scope `read`) returns historical runs newest-first.

**Acceptance criteria**:

- **AC-2.1**: `POST /v1/tasks/{id}/run` with a `write` key returns **HTTP 202** with a `run_id` in the body. Verified by `tests/.../test_fr02_run_returns_202`. Per SPEC.md §3 FR-02.
- **AC-2.2**: The runner calls `asyncio.create_subprocess_exec(*shlex.split(command))`; `shell=True` is never passed. Verified by `tests/.../test_fr02_runner_uses_subprocess_exec_no_shell` (introspects `subprocess.Popen` arguments). Per SPEC.md §3 FR-02 and NFR-02.
- **AC-2.3**: When the command exceeds `TASKQ_TASK_TIMEOUT` seconds, the task state transitions to `timeout` and `exit_code`, `stdout_tail`, `stderr_tail`, `duration_ms`, `finished_at` are persisted to `task_results`. Verified by `tests/.../test_fr02_timeout_records_results`. Per SPEC.md §3 FR-02.
- **AC-2.4**: `GET /v1/tasks/{id}/runs` returns the task's runs newest-first, and is restricted to `read` scope. Verified by `tests/.../test_fr02_runs_newest_first` and `test_fr02_runs_requires_read_scope`. Per SPEC.md §3 FR-02.

> DERIVED: SPEC.md §3 FR-03 (lines 101–108) + §8 #18 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### FR-03: API Key 認證

Canonical source: SPEC.md §3 FR-03 (lines 101–108).

- Every `/v1/*` endpoint requires the `X-API-Key` header; missing or invalid key → **HTTP 401** + problem+json.
- Keys are stored as **SHA-256 hashes** in the `api_keys` table; plaintext is NEVER stored; comparison uses `hmac.compare_digest` (constant-time).
- Keys are created via `python -m taskq_api key create --scope <scope>`; the plaintext is printed exactly once at creation.
- A key whose `revoked_at` is non-null is treated as invalid.
- `/healthz` and `/readyz` do not require authentication.

**Acceptance criteria**:

- **AC-3.1**: Any `/v1/*` request missing the `X-API-Key` header returns **HTTP 401** + problem+json (`type=/errors/unauthenticated`). Verified by `tests/.../test_fr03_missing_api_key_returns_401`. Per SPEC.md §8 #5.
- **AC-3.2**: Any `/v1/*` request with an unknown or revoked `X-API-Key` returns **HTTP 401** + problem+json. Verified by `tests/.../test_fr03_invalid_or_revoked_key_returns_401`. Per SPEC.md §3 FR-03.
- **AC-3.3**: The `api_keys` table contains ONLY SHA-256 hashes (`key_hash` is 64 hex chars); no plaintext keys are stored anywhere in the database. Verified by `tests/.../test_fr03_no_plaintext_keys_in_db`. Per SPEC.md §8 #18 and NFR-02.
- **AC-3.4**: Key comparison uses `hmac.compare_digest` (constant-time). Verified by `tests/.../test_fr03_comparison_uses_hmac_compare_digest` (introspects the auth dependency). Per SPEC.md §3 FR-03.
- **AC-3.5**: `python -m taskq_api key create --scope <scope>` outputs the plaintext key once at creation and never persists it. Verified by `tests/.../test_fr03_key_create_prints_plaintext_once`. Per SPEC.md §3 FR-03.
- **AC-3.6**: `GET /healthz` and `GET /readyz` accept requests with no `X-API-Key` header (FR-09). Verified by `tests/.../test_fr03_health_endpoints_no_auth`. Per SPEC.md §3 FR-03.

> DERIVED: SPEC.md §3 FR-04 (lines 109–114) + §8 #6 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### FR-04: Scope 授權

Canonical source: SPEC.md §3 FR-04 (lines 109–114).

- Each key carries a scope: `read` < `write` < `admin` (hierarchical inclusion).
- Per-endpoint required scope is the FR-01/02 table; insufficient → **HTTP 403** + problem+json; the body MUST NOT disclose whether the resource exists.
- Authorization MUST be enforced via a single middleware/dependency; the test asserts "every `/v1` route passes through the same dependency".

**Acceptance criteria**:

- **AC-4.1**: `DELETE /v1/tasks/{id}` with a `write` (non-admin) key returns **HTTP 403**; the body MUST NOT disclose the resource's existence. Verified by `tests/.../test_fr04_scope_insufficient_returns_403_no_leak`. Per SPEC.md §8 #6 and FR-04.
- **AC-4.2**: Every `/v1/*` route passes through exactly the same scope-dependency function. Verified by `tests/.../test_fr04_all_v1_routes_share_single_scope_dependency` (introspects FastAPI `app.routes` and asserts a single dependency is attached). Per SPEC.md §3 FR-04.
- **AC-4.3**: Scope hierarchy: an `admin` key passes `read`-required and `write`-required checks; a `read` key does NOT pass `write`-required checks. Verified by `tests/.../test_fr04_scope_hierarchy_inclusion`. Per SPEC.md §3 FR-04.

> DERIVED: SPEC.md §3 FR-05 (lines 115–121) + §8 #9 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### FR-05: 流量控制

Canonical source: SPEC.md §3 FR-05 (lines 115–121).

- Per-token token bucket: capacity `TASKQ_RATE_BURST`, refill rate `TASKQ_RATE_PER_SEC`.
- Over-limit → **HTTP 429** + problem+json + `Retry-After` header (seconds).
- Bucket state lives in the database (cross-worker consistency); updates MUST run inside a single transaction with row-level lock.
- `/healthz` and `/readyz` are NOT rate-limited.

**Acceptance criteria**:

- **AC-5.1**: Bursting requests beyond `TASKQ_RATE_BURST` against one token return **HTTP 429** + problem+json + a `Retry-After` header. Verified by `tests/.../test_fr05_rate_limit_burst_returns_429_with_retry_after`. Per SPEC.md §8 #9.
- **AC-5.2**: The rate bucket update runs inside a single transaction with row-level lock (no double-spend across concurrent workers). Verified by `tests/.../test_fr05_rate_bucket_row_level_lock_no_double_spend`. Per SPEC.md §3 FR-05.
- **AC-5.3**: `GET /healthz` and `GET /readyz` are exempt from the rate limit. Verified by `tests/.../test_fr05_health_endpoints_not_rate_limited`. Per SPEC.md §3 FR-05.

> DERIVED: SPEC.md §3 FR-06 (lines 122–129) + §8 #14 #17 #21 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### FR-06: 持久化層與交易邊界

Canonical source: SPEC.md §3 FR-06 (lines 122–129).

- All data access goes through the `repository/` layer; business layers MUST NOT directly hold a `Session`.
- One `Session` per API request; explicit transaction boundaries: commit on success, rollback on exception (enforced by context manager).
- String-concatenated SQL is forbidden; ORM or parameterized queries only (NFR-02).
- Eager loading via `selectinload` / `joinedload` is required — **N+1 is an acceptance failure condition** (NFR-01).
- Connection pool: `pool_size=TASKQ_DB_POOL_SIZE`, `pool_pre_ping=True`.

**Acceptance criteria**:

- **AC-6.1**: One `Session` is opened per API request and closed at request end (commit on success / rollback on exception). Verified by `tests/.../test_fr06_session_per_request_commit_rollback`. Per SPEC.md §3 FR-06 and NFR-03.
- **AC-6.2**: The `service/` and `api/` layers do NOT import `sqlalchemy` directly. Verified by `lint-imports` exit 0 + the `repository` forbidden-imports contract. Per SPEC.md §8 #21 and NFR-06.
- **AC-6.3**: List endpoints issue a CONSTANT number of SQL statements regardless of result count (no N+1). Verified by `tests/.../test_fr06_list_endpoint_constant_sql_count` (SQLAlchemy event listener). Per SPEC.md §8 #14 and NFR-01.
- **AC-6.4**: SQL string-concatenation patterns (f-string / `%` / `+` building SQL) return **0 hits** in `03-development/src/`. Verified by `tests/.../test_fr06_no_sql_string_concat` (grep gate). Per SPEC.md §8 #17 and NFR-02.
- **AC-6.5**: The connection pool uses `pool_size=TASKQ_DB_POOL_SIZE` and `pool_pre_ping=True`. Verified by `tests/.../test_fr06_pool_config`. Per SPEC.md §3 FR-06.

> DERIVED: SPEC.md §3 FR-07 (lines 130–144) + §8 #11-#13 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim; v3 data-migration payload shape is OI-1 (deferred, NFR-99 escape).

### FR-07: Schema Migration (Alembic 三步演進)

Canonical source: SPEC.md §3 FR-07 (lines 130–144).

Three revisions, each with a working `downgrade`:

| revision | upgrade | downgrade requirement |
|---|---|---|
| **v1** | create `tasks`, `api_keys` | drop both tables |
| **v2** | add `tags`, `task_tags` (many-to-many) + unique index on `tasks.name` | drop new tables and index; do not touch v1 data |
| **v3** | **data migration**: split `tasks.result_json` into a separate `task_results` table; migrate existing data then drop the original column | reverse-migrate back into `tasks.result_json` then drop `task_results`; **no data loss** |

- `alembic upgrade head` and `alembic downgrade base` MUST both succeed.
- **Round-trip reversibility acceptance**: `upgrade head` → write sample data → `downgrade -1` → `upgrade head`; the sample data's columns MUST match field-by-field (v3 data migration is the focus of this clause).
- Destructive shortcuts like `op.execute("DROP TABLE ...")` are forbidden in place of a real downgrade.
- Migration files themselves are under test coverage (using `alembic` offline SQL generation + assertions).

**Acceptance criteria**:

- **AC-7.1**: `alembic upgrade head` exits 0 on a clean SQLite file. Verified by `tests/.../test_fr07_alembic_upgrade_head`. Per SPEC.md §3 FR-07.
- **AC-7.2**: `alembic downgrade base` exits 0 with no residual tables after `upgrade head`. Verified by `tests/.../test_fr07_alembic_downgrade_base_no_residue`. Per SPEC.md §8 #13.
- **AC-7.3**: v3 round-trip on a real SQLite file: `upgrade head` → seed sample rows with non-trivial `result_json` payloads → `downgrade -1` → `upgrade head`; every column value matches field-by-field. Verified by `tests/.../test_fr07_v3_round_trip_field_by_field`. Per SPEC.md §8 #12 and NFR-09.
- **AC-7.4**: No `op.execute("DROP TABLE ...")` shortcut replaces a real downgrade. Verified by `tests/.../test_fr07_no_drop_table_shortcut` (grep over `migrations/versions/`). Per SPEC.md §3 FR-07.
- **AC-7.5**: Each revision file has a working `downgrade()` function callable from `alembic`. Verified by `tests/.../test_fr07_each_revision_has_working_downgrade`. Per SPEC.md §3 FR-07.
- **AC-7.6**: After `alembic downgrade -1` from head, `GET /readyz` returns **HTTP 503** with `detail` indicating "migration not at head". Verified by `tests/.../test_fr07_readyz_fails_after_downgrade`. Per SPEC.md §8 #11 and FR-09.

> DERIVED: SPEC.md §3 FR-08 (lines 145–151) + §8 #25 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### FR-08: 非同步執行器

Canonical source: SPEC.md §3 FR-08 (lines 145–151).

- Background execution is managed by `asyncio.TaskGroup`; on shutdown the service MUST **graceful-drain** (wait for in-flight tasks up to `TASKQ_DRAIN_TIMEOUT`; tasks still running past the timeout are marked `interrupted`).
- Concurrency cap `TASKQ_MAX_CONCURRENT`; requests beyond the cap queue rather than spawning unbounded coroutines.
- Task timeout via `asyncio.wait_for`; on timeout the child process MUST be terminated (`process.kill()` then `await process.wait()`); orphan processes are forbidden.
- `asyncio.CancelledError` MUST propagate upward; it MUST NOT be swallowed by `except Exception:` (NFR-03).

**Acceptance criteria**:

- **AC-8.1**: Service shutdown with in-flight tasks waits up to `TASKQ_DRAIN_TIMEOUT`; tasks finishing in time are not marked `interrupted`; tasks exceeding the cap are marked `interrupted`. Verified by `tests/.../test_fr08_graceful_drain_marks_interrupted`. Per SPEC.md §8 #25 and NFR-03.
- **AC-8.2**: When concurrent in-flight task count >= `TASKQ_MAX_CONCURRENT`, additional runs queue rather than spawning unbounded coroutines. Verified by `tests/.../test_fr08_concurrency_cap_queues_excess`. Per SPEC.md §3 FR-08.
- **AC-8.3**: On task timeout, the subprocess is killed (`process.kill()` then `await process.wait()`); zero orphan subprocesses remain. Verified by `tests/.../test_fr08_timeout_kills_subprocess_no_orphans`. Per SPEC.md §3 FR-08 and §8 #25.
- **AC-8.4**: `asyncio.CancelledError` is re-raised, not swallowed by `except Exception:`. Verified by `tests/.../test_fr08_cancelled_error_propagates`. Per SPEC.md §3 FR-08 and NFR-03.

> DERIVED: SPEC.md §3 FR-09 (lines 152–161) + §8 #10 #11 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### FR-09: 健康檢查與可觀測性

Canonical source: SPEC.md §3 FR-09 (lines 152–161).

| endpoint | auth | behavior |
|------|------|------|
| `GET /healthz` | none | process alive → 200 `{"status":"ok"}` |
| `GET /readyz` | none | DB connection usable **AND** `alembic current` == head → 200; otherwise **503** with the failing item named in body |
| `GET /v1/metrics` | `admin` | task counts (by status), execution-latency percentiles, rate-limit rejection counts |

- The "/readyz" fail-closed on "migration not at head" judgement is critical: shipping new code without running migrations MUST fail closed.

**Acceptance criteria**:

- **AC-9.1**: `GET /healthz` returns **HTTP 200** with body `{"status":"ok"}` when the process is alive, regardless of DB state. Verified by `tests/.../test_fr09_healthz_returns_200`. Per SPEC.md §3 FR-09.
- **AC-9.2**: `GET /readyz` returns **HTTP 200** when DB is reachable AND `alembic current == head`. Verified by `tests/.../test_fr09_readyz_returns_200_when_ready`. Per SPEC.md §3 FR-09.
- **AC-9.3**: `GET /readyz` returns **HTTP 503** with `detail` indicating "DB unavailable" when the DB is down. Verified by `tests/.../test_fr09_readyz_503_when_db_down`. Per SPEC.md §8 #10.
- **AC-9.4**: `GET /readyz` returns **HTTP 503** with `detail` indicating "migration not at head" after `alembic downgrade -1` is run. Verified by `tests/.../test_fr09_readyz_503_when_migration_behind`. Per SPEC.md §8 #11 and FR-07.
- **AC-9.5**: `GET /v1/metrics` requires `admin` scope; returns task counts by status, execution-latency percentiles, and rate-limit rejection counts. Verified by `tests/.../test_fr09_metrics_endpoint_requires_admin_and_returns_payload`. Per SPEC.md §3 FR-09.

> DERIVED: SPEC.md §3 FR-10 (lines 162–171) + §8 #19 + §7 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### FR-10: 錯誤契約 (RFC 7807)

Canonical source: SPEC.md §3 FR-10 (lines 162–171).

- All non-2xx responses have `Content-Type: application/problem+json`.
- Body fields: `type` (URI), `title`, `status`, `detail`, `instance`, `correlation_id`.
- **`detail` MUST NOT leak internal details**: no SQL statements, no stack traces, no file paths, no DB schema.
- `correlation_id` appears in both the response header `X-Correlation-Id` and the server log.
- Error code mapping: 422 validation / 401 unauthenticated / 403 insufficient scope / 404 unknown resource / 409 name conflict / 429 rate-limited / 503 not ready / 500 other.

**Acceptance criteria**:

- **AC-10.1**: Every non-2xx response has `Content-Type: application/problem+json`. Verified by `tests/.../test_fr10_problem_json_content_type`. Per SPEC.md §3 FR-10.
- **AC-10.2**: Every problem+json body contains `type`, `title`, `status`, `detail`, `instance`, `correlation_id` fields. Verified by `tests/.../test_fr10_problem_json_required_fields`. Per SPEC.md §3 FR-10.
- **AC-10.3**: Forcing a 500 internal error, the response `detail` contains no stack trace, no SQL, no file path. Verified by `tests/.../test_fr10_500_detail_redacts_internal`. Per SPEC.md §8 #19 and NFR-02.
- **AC-10.4**: Every error response also carries `correlation_id` in the `X-Correlation-Id` response header, and that same id appears in server log output. Verified by `tests/.../test_fr10_correlation_id_in_header_and_log`. Per SPEC.md §3 FR-10.
- **AC-10.5**: Error code mapping per SPEC.md §7 table is implemented: 422/401/403/404/409/429/503/500 each fire under the documented condition. Verified by `tests/.../test_fr10_error_code_mapping`. Per SPEC.md §7 and FR-10.

## 4. Non-Functional Requirements (NFRs)

Each NFR below transcribes one `### NFR-XX` heading from canonical SPEC.md verbatim. Each AC carries a stable identifier of the form `AC-N<n>.<m>`.

> DERIVED: SPEC.md §4 NFR-01 (lines 177–184) + §8 #14 #15 + §11 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim; "常數" ambiguity resolved to fixture-pair constant via AC-N1.3.

### NFR-01: 效能與查詢效率

Canonical source: SPEC.md §4 NFR-01 (lines 177–184). **dimension**: `performance` (confirmed in current `evaluate_dimension.md` Tier-3 roster as `performance`).

- `GET /v1/tasks/{id}` over 10,000 rows: **p95 < 30ms** (excluding network, ASGI transport).
- `GET /v1/tasks?limit=50` over 10,000 rows: **p95 < 80ms**.
- **N+1 is a failure condition**: the number of SQL statements issued by a list-endpoint request MUST be **constant** (independent of result count), asserted via a SQLAlchemy event listener.
- Measurement: `pytest-benchmark`.

**Acceptance criteria**:

- **AC-N1.1**: `pytest-benchmark` reports `GET /v1/tasks/{id}` p95 latency < 30ms over a 10,000-row fixture. Verified by `tests/.../test_nfr01_get_task_p95_under_30ms`. Per SPEC.md §8 #15 and §11.
- **AC-N1.2**: `pytest-benchmark` reports `GET /v1/tasks?limit=50` p95 latency < 80ms over a 10,000-row fixture. Verified by `tests/.../test_nfr01_list_tasks_p95_under_80ms`. Per SPEC.md §11.
- **AC-N1.3**: A SQLAlchemy event listener counts SQL statements emitted during a `GET /v1/tasks?limit=50` call; the count is constant across fixture sizes (e.g. 100-row fixture and 1000-row fixture both produce ≤ C statements where C is the constant bound measured at 100 rows). Verified by `tests/.../test_nfr01_list_endpoint_constant_sql_count`. Per SPEC.md §8 #14 and §11.

> DERIVED: SPEC.md §4 NFR-02 (lines 185–195) + §8 #16 #17 #18 #19 #23 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### NFR-02: HTTP 與資料層安全

Canonical source: SPEC.md §4 NFR-02 (lines 185–195). **dimension**: `security` (confirmed in current `evaluate_dimension.md` Tier-2 roster as `security`).

- All `shell=True`, `eval(`, `exec(` are forbidden across the codebase (grep = 0 hits).
- SQL string concatenation is forbidden: no f-string / `%` / `+` building SQL; ORM or parameterized only (verified via grep + code review).
- API keys stored hashed, compared via `hmac.compare_digest` (FR-03).
- 403 responses MUST NOT disclose resource existence (FR-04).
- Error bodies MUST NOT contain stack/SQL/path (FR-10).
- CORS by default rejects ALL origins; allowlist is `TASKQ_CORS_ORIGINS`.
- `bandit -r 03-development/src/`: **0 HIGH, 0 MEDIUM**.

**Acceptance criteria**:

- **AC-N2.1**: `grep -rn "shell=True\|eval(\|exec(" 03-development/src/` returns **0 hits**. Verified by `tests/.../test_nfr02_no_shell_eval_exec`. Per SPEC.md §8 #16.
- **AC-N2.2**: `bandit -r 03-development/src/` returns **0 HIGH, 0 MEDIUM** findings. Verified by `tests/.../test_nfr02_bandit_clean`. Per SPEC.md §8 #23.
- **AC-N2.3**: CORS middleware with default policy rejects all origins unless `TASKQ_CORS_ORIGINS` lists them. Verified by `tests/.../test_nfr02_cors_default_deny_with_allowlist`. Per SPEC.md §3 FR-02 / NFR-02.

> DERIVED: SPEC.md §4 NFR-03 (lines 196–205) + §8 #10 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### NFR-03: 錯誤處理、交易與非同步正確性

Canonical source: SPEC.md §4 NFR-03 (lines 196–205). **dimension**: `reliability` (matches the FR Block `type: reliability` declaration and the pinned vocabulary declared in this document, line 610).

- Each request has explicit transaction boundaries: commit on success / rollback on exception, enforced via context manager (FR-06).
- Bare `except:` and `except Exception: pass` are forbidden.
- `asyncio.CancelledError` MUST NOT be swallowed — it MUST be re-raised (the async-specific swallow trap).
- DB connection failure → `/readyz` 503 with explicit detail; infinite silent retries are forbidden.
- Task timeout MUST actually terminate the subprocess; no orphans (FR-08).
- Migration failure → transaction rollback; DB remains at the previous revision (FR-07).

**Acceptance criteria**:

- **AC-N3.1**: Static AST check finds no bare `except:` and no `except Exception: pass` patterns in `03-development/src/`. Verified by `tests/.../test_nfr03_no_bare_except`. Per SPEC.md §3 NFR-03.
- **AC-N3.2**: `asyncio.CancelledError` raised inside a runner coroutine propagates out (test triggers cancellation and asserts it reaches the request task). Verified by `tests/.../test_nfr03_cancelled_error_propagates`. Per SPEC.md §3 NFR-03.
- **AC-N3.3**: DB connection failure surfaces as `/readyz` 503 with `detail` describing DB unavailable; the service does NOT silently retry forever. Verified by `tests/.../test_nfr03_db_failure_readyz_503_no_infinite_retry`. Per SPEC.md §3 NFR-03 and §8 #10.
- **AC-N3.4**: Migration failure during a request rolls back the transaction; the DB remains at the previous revision. Verified by `tests/.../test_nfr03_migration_failure_rolls_back`. Per SPEC.md §3 NFR-03 and FR-07.

> DERIVED: SPEC.md §4 NFR-04 (lines 206–213) + §8 #20 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### NFR-04: 敏感資料遮蔽

Canonical source: SPEC.md §4 NFR-04 (lines 206–213). **dimension**: `security`.

- `stdout_tail` / `stderr_tail` / logs / error bodies are scanned before persistence or egress; lines matching `(sk-[A-Za-z0-9_-]{8,}|token=\S+|Bearer\s+\S+|postgres(ql)?://[^\s]+)` are replaced with `[REDACTED]`.
- DB connection strings (including passwords) MUST NOT appear in any log, error message, or `/v1/metrics` response.
- API key plaintext is output exactly once at `key create`; it is NEVER written to any persistent location.

**Acceptance criteria**:

- **AC-N4.1**: A `stdout_tail` containing a string matching `sk-[A-Za-z0-9_-]{8,}` is persisted with that line replaced by `[REDACTED]`. Verified by `tests/.../test_nfr04_redact_sk_token_in_stdout`. Per SPEC.md §3 NFR-04.
- **AC-N4.2**: DB connection strings with passwords do not appear in any log line, error body, or `/v1/metrics` response. Verified by `tests/.../test_nfr04_no_db_url_in_logs_or_metrics`. Per SPEC.md §8 #20 and §11.
- **AC-N4.3**: API key plaintext is printed only at `key create`; subsequent queries to `api_keys` show only the hash. Verified by `tests/.../test_nfr04_key_plaintext_only_at_create`. Per SPEC.md §3 NFR-04 and FR-03.

> DERIVED: SPEC.md §4 NFR-05 (lines 214–219) — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### NFR-05: 文件覆蓋

Canonical source: SPEC.md §4 NFR-05 (lines 214–219). **dimension**: `documentation`.

- All public functions/classes have a docstring that includes `[FR-XX]` or `[NFR-XX]` references; coverage **100%**.
- Each API endpoint has `summary` and `description` in the OpenAPI schema (verified by assertions on the `/openapi.json` produced by FastAPI).

**Acceptance criteria**:

- **AC-N5.1**: AST check reports 100% of public functions/classes in `03-development/src/` carry a docstring referencing at least one `[FR-XX]` or `[NFR-XX]` token. Verified by `tests/.../test_nfr05_docstring_coverage_100_percent`. Per SPEC.md §3 NFR-05.
- **AC-N5.2**: `GET /openapi.json` lists every `/v1/*` endpoint with non-empty `summary` and `description` fields. Verified by `tests/.../test_nfr05_openapi_summary_description`. Per SPEC.md §3 NFR-05.

> DERIVED: SPEC.md §4 NFR-06 (lines 220–233) + §8 #21 + §11 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### NFR-06: 架構分層契約

Canonical source: SPEC.md §4 NFR-06 (lines 220–233). **dimension**: `architecture_constraints` (confirmed in current `evaluate_dimension.md` Tier-1 roster as `architecture_constraints`).

- A `.importlinter` file MUST exist at the project root declaring the layers contract: `api > service > repository > models`. Upper layers may import lower layers; lower layers MUST NOT import upper layers. `config` and `errors` are independence modules.
- **Forbidden contract (additional)**: any layer other than `repository` MUST NOT import `sqlalchemy` — ORM leakage into the business layer is the specific anti-pattern this round defends against.
- `lint-imports` MUST exit 0.
- Removing `.importlinter`, wildcard `ignore_imports`, or downgrading the contract to pass are all forbidden.

**Acceptance criteria**:

- **AC-N6.1**: `.importlinter` exists at the project root and declares `api > service > repository > models`. Verified by `tests/.../test_nfr06_importlinter_exists_with_layers_contract`. Per SPEC.md §3 NFR-06.
- **AC-N6.2**: `lint-imports` exits 0; introducing a forbidden import (e.g. `from sqlalchemy import …` in `service/`) makes `lint-imports` exit non-zero. Verified by `tests/.../test_nfr06_lint_imports_exit_0_and_blocks_sqlalchemy`. Per SPEC.md §8 #21 and §11.
- **AC-N6.3**: The `.importlinter` file is NOT deleted or downgraded; a check that the contract file exists with the four-layer declaration is part of CI. Verified by `tests/.../test_nfr06_importlinter_not_removed`. Per SPEC.md §3 NFR-06.

> DERIVED: SPEC.md §4 NFR-07 (lines 234–241) + §8 #22 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### NFR-07: 依賴與授權合規

Canonical source: SPEC.md §4 NFR-07 (lines 234–241). **dimension**: `license_compliance` (confirmed in current `evaluate_dimension.md` Tier-1 roster as `license_compliance`).

- All runtime dependencies pinned with `==` in `requirements.txt`; transitive dependencies fully locked via `requirements.lock`.
- Allowed licenses: MIT, BSD-2-Clause, BSD-3-Clause, Apache-2.0, PSF. Any other license → that dependency MUST NOT be used.
- **Scan scope MUST include the full dependency tree** (direct + transitive); evidence command: `pip-licenses --format=json --with-system`.
- SBOM produced at `08-config/SBOM.json` containing each dependency's `name` / `version` / `license` / `direct|transitive`.

**Acceptance criteria**:

- **AC-N7.1**: `pip-licenses --format=json --with-system` reports every dependency (direct + transitive) with `license` ∈ {MIT, BSD-2-Clause, BSD-3-Clause, Apache-2.0, PSF}. Verified by `tests/.../test_nfr07_pip_licenses_all_in_allowlist`. Per SPEC.md §8 #22.
- **AC-N7.2**: `08-config/SBOM.json` exists and contains entries for every direct + transitive dependency with `name`, `version`, `license`, `direct|transitive`. Verified by `tests/.../test_nfr07_sbom_artifact_present`. Per SPEC.md §3 NFR-07.
- **AC-N7.3**: `requirements.txt` pins every runtime dep with `==`; `requirements.lock` exists and locks transitive deps. Verified by `tests/.../test_nfr07_requirements_pinned_and_lockfile_present`. Per SPEC.md §3 NFR-07.

> DERIVED: SPEC.md §4 NFR-08 (lines 242–248) + §8 #24 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim; budget figure deferred via OI-3 NFR-99 escape.

### NFR-08: 變異測試

Canonical source: SPEC.md §4 NFR-08 (lines 242–248). **dimension**: `mutation_testing` (confirmed in current `evaluate_dimension.md` Tier-1 roster as `mutation_testing`).

- `.methodology/harness_config.json` sets `features.mutation_testing: true`.
- **mutation score ≥ 70**.
- Scope is limited to `service/` and `repository/` and the limitation rationale is noted in `harness_config.json` (execution-time budget).

**Acceptance criteria**:

- **AC-N8.1**: `mutmut run` followed by `mutmut results` reports a mutation score ≥ 70. Verified by `tests/.../test_nfr08_mutation_score_at_least_70`. Per SPEC.md §8 #24.
- **AC-N8.2**: `.methodology/harness_config.json` contains `features.mutation_testing: true`. Verified by `tests/.../test_nfr08_harness_config_mutation_testing_enabled`. Per SPEC.md §3 NFR-08.

> DERIVED: SPEC.md §4 NFR-09 (lines 249–258) + §8 #1 + §11 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### NFR-09: 驗證真實性 (零 skip 鐵律)

Canonical source: SPEC.md §4 NFR-09 (lines 249–258). **dimension**: `test_assertion_quality` (confirmed in current `evaluate_dimension.md` Tier-2 roster as `test_assertion_quality`).

- **No FR/NFR verification test may be `pytest.skip` / `skipif` / `xfail` / an assertion-less stub.**
- `pytest 03-development/tests -q` `skipped` count MUST be **0**.
- Each test function has at least one `assert` (`zero_assert == 0`).
- **Anti-fabrication clause**: excluding tests via `--ignore` / `-k` / `--deselect` / `collect_ignore` / removing a directory from `testpaths` is forbidden.
- **Round-2 specific clause**: FR-07's three-step migration MUST be tested against a **real database** (SQLite file, NOT in-memory mock); round-trip reversibility is verified by actual data comparison. The "migration logic is too hard to test" downgrade-to-skip is forbidden — this is the failure shape of the prior two rounds.
- `TRACEABILITY_MATRIX.md` `VERIFIED` is granted ONLY when the test actually ran and passed.

**Acceptance criteria**:

- **AC-N9.1**: `pytest 03-development/tests -q` exit code is 0 and the `skipped` summary count is **0**. Verified by `tests/.../test_nfr09_zero_skipped`. Per SPEC.md §8 #1 and §11.
- **AC-N9.2**: AST scan finds zero test functions with no `assert` calls. Verified by `tests/.../test_nfr09_zero_assertion_less_tests`. Per SPEC.md §3 NFR-09 and §11.
- **AC-N9.3**: FR-07 migration tests run against a real SQLite file (not in-memory `:memory:`). Verified by `tests/.../test_nfr09_migration_tests_use_real_sqlite_file`. Per SPEC.md §3 NFR-09.
- **AC-N9.4**: `TRACEABILITY_MATRIX.md` `VERIFIED` cells correspond only to tests that actually ran and passed; a duplicated `VERIFIED` entry without a runnable test is rejected by a check. Verified by `tests/.../test_nfr09_traceability_verified_only_for_actual_runs`. Per SPEC.md §3 NFR-09.

> DERIVED: SPEC.md §4 NFR-10 (lines 259–265) + §8 #3 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### NFR-10: 整合覆蓋

Canonical source: SPEC.md §4 NFR-10 (lines 259–265). **dimension**: `integration_coverage` (confirmed in current `evaluate_dimension.md` Tier-2 roster as `integration_coverage`).

- `03-development/tests/integration/` line coverage **≥ 80%**.
- Integration tests are driven by `httpx.AsyncClient(transport=ASGITransport(app))`; calling handler functions directly is forbidden.
- Must cover at minimum: full CRUD chain, each of 401/403/404/409/422/429/503 with one example, migration round-trip, rate-limit triggering and recovery, graceful drain.

**Acceptance criteria**:

- **AC-N10.1**: `pytest 03-development/tests/integration --cov=03-development/src --cov-report=term` reports **TOTAL ≥ 80%**. Verified by `tests/.../test_nfr10_integration_coverage_at_least_80_percent`. Per SPEC.md §8 #3.
- **AC-N10.2**: No integration test imports a handler function directly; every integration test goes through `httpx.AsyncClient(transport=ASGITransport(app))`. Verified by `tests/.../test_nfr10_integration_uses_asgi_transport_not_direct_call`. Per SPEC.md §3 NFR-10.
- **AC-N10.3**: Integration suite contains at least one example for each error code 401/403/404/409/422/429/503. Verified by `tests/.../test_nfr10_each_error_code_covered`. Per SPEC.md §3 NFR-10.

> DERIVED: SPEC.md §4 NFR-11 (lines 266–272) — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim; FR Block `type` mapping `maintainability` per pinned vocabulary (vocabulary list omits `readability`).

### NFR-11: 可讀性

Canonical source: SPEC.md §4 NFR-11 (lines 266–272). **dimension**: `readability` (confirmed in current `evaluate_dimension.md` Tier-3 roster as `readability`). FR Block `type` mapping: `maintainability` (vocabulary list omits `readability`).

- Project MI (LLOC-weighted) ≥ 80; per-function CC ≤ 10.
- Single file ≤ 400 lines; single directory ≤ 15 files.
- Each API handler ≤ 40 lines (business logic MUST be pushed down to `service/`).

**Acceptance criteria**:

- **AC-N11.1**: Readability tool (radon-mi / readability-v2) reports project MI ≥ 80. Verified by `tests/.../test_nfr11_mi_at_least_80`. Per SPEC.md §8 and §11.
- **AC-N11.2**: No function exceeds CC 10; no file exceeds 400 lines; no directory exceeds 15 files. Verified by `tests/.../test_nfr11_size_and_cc_thresholds`. Per SPEC.md §3 NFR-11.
- **AC-N11.3**: No API handler function exceeds 40 lines; business logic is in `service/`. Verified by `tests/.../test_nfr11_handlers_under_40_lines`. Per SPEC.md §3 NFR-11.

> DERIVED: SPEC.md §4 NFR-12 (lines 273–284) + §8 #27 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

### NFR-12: 系統驗證目標

> DERIVED: SPEC.md §4 NFR-12 (lines 273–284) + §8 #27 — English summary is a derivative paraphrase; bullet ACs use canonical phrasing verbatim.

Canonical source: SPEC.md §4 NFR-12 (lines 273–284). **dimension**: `execute_verification_target` (confirmed in current `evaluate_dimension.md` Tier-1 roster as `execute_verification_target`).

- `Makefile` `verify-system` target chains:
  1. `alembic upgrade head`
  2. full test suite
  3. service startup + `/healthz`, `/readyz` smoke
  4. `alembic downgrade base` then `upgrade head` (round-trip verification)
- `make verify-system` MUST exit 0 and print `verify-system: PASS` on stdout.

**Acceptance criteria**:

- **AC-N12.1**: `make verify-system` exits 0 and stdout contains the literal `verify-system: PASS`. Verified by `tests/.../test_nfr12_make_verify_system_passes`. Per SPEC.md §8 #27.
- **AC-N12.2**: The `Makefile` target chains all four steps (upgrade head → tests → startup smoke → downgrade base → upgrade head). Verified by `tests/.../test_nfr12_verify_system_includes_migration_round_trip`. Per SPEC.md §3 NFR-12.

## 5. Acceptance Criteria Summary

This section is a flat index of every `AC-x.y` / `AC-Nx.y` defined above, organized by FR/NFR. Each entry references the SPEC.md citation.

| ID | Description | SPEC.md citation | Verification function |
|----|-------------|------------------|------------------------|
| AC-1.1 | POST /v1/tasks valid body → 201 | §8 #4 | test_fr01_post_returns_201 |
| AC-1.2 | POST /v1/tasks missing key → 401 | §8 #5 | test_fr01_post_no_api_key_returns_401 |
| AC-1.3 | POST /v1/tasks invalid body → 422/409 | §3 FR-01 | test_fr01_post_invalid_body_returns_422_or_409 |
| AC-1.4 | POST /v1/tasks duplicate name → 409 | §8 #8 | test_fr01_post_duplicate_name_returns_409 |
| AC-1.5 | GET /v1/tasks/{unknown} → 404 | §8 #7 | test_fr01_get_unknown_returns_404 |
| AC-1.6 | GET /v1/tasks cursor-based; limit cap 200 | §3 FR-01 | test_fr01_list_uses_cursor_pagination · test_fr01_list_limit_above_200_returns_422 |
| AC-1.7 | DELETE /v1/tasks/{id} with write key → 403, no existence leak | §8 #6 | test_fr01_delete_non_admin_returns_403_no_existence_leak |
| AC-1.8 | DELETE /v1/tasks/{id} admin: atomic | §3 FR-01 | test_fr01_delete_admin_atomic |
| AC-2.1 | POST /v1/tasks/{id}/run → 202 | §3 FR-02 | test_fr02_run_returns_202 |
| AC-2.2 | Runner uses subprocess_exec, no shell=True | §3 FR-02 | test_fr02_runner_uses_subprocess_exec_no_shell |
| AC-2.3 | Timeout → records results, status=timeout | §3 FR-02 | test_fr02_timeout_records_results |
| AC-2.4 | GET /v1/tasks/{id}/runs newest-first, read scope | §3 FR-02 | test_fr02_runs_newest_first · test_fr02_runs_requires_read_scope |
| AC-3.1 | /v1/* missing API key → 401 | §8 #5 | test_fr03_missing_api_key_returns_401 |
| AC-3.2 | /v1/* invalid/revoked key → 401 | §3 FR-03 | test_fr03_invalid_or_revoked_key_returns_401 |
| AC-3.3 | api_keys has only hashes (64 hex) | §8 #18 | test_fr03_no_plaintext_keys_in_db |
| AC-3.4 | Comparison uses hmac.compare_digest | §3 FR-03 | test_fr03_comparison_uses_hmac_compare_digest |
| AC-3.5 | key create prints plaintext once | §3 FR-03 | test_fr03_key_create_prints_plaintext_once |
| AC-3.6 | /healthz, /readyz need no auth | §3 FR-03 | test_fr03_health_endpoints_no_auth |
| AC-4.1 | DELETE with write key → 403, no existence leak | §8 #6 | test_fr04_scope_insufficient_returns_403_no_leak |
| AC-4.2 | Single scope dependency on every /v1/* | §3 FR-04 | test_fr04_all_v1_routes_share_single_scope_dependency |
| AC-4.3 | Scope hierarchy inclusion | §3 FR-04 | test_fr04_scope_hierarchy_inclusion |
| AC-5.1 | Burst → 429 + Retry-After | §8 #9 | test_fr05_rate_limit_burst_returns_429_with_retry_after |
| AC-5.2 | Rate bucket row-level lock | §3 FR-05 | test_fr05_rate_bucket_row_level_lock_no_double_spend |
| AC-5.3 | /healthz, /readyz not rate-limited | §3 FR-05 | test_fr05_health_endpoints_not_rate_limited |
| AC-6.1 | Session per request, commit/rollback via ctx mgr | §3 FR-06 | test_fr06_session_per_request_commit_rollback |
| AC-6.2 | service/api layers do not import sqlalchemy | §8 #21 | test_fr06_no_sqlalchemy_in_service_or_api |
| AC-6.3 | List endpoint constant SQL count | §8 #14 | test_fr06_list_endpoint_constant_sql_count |
| AC-6.4 | No SQL string concat (grep 0) | §8 #17 | test_fr06_no_sql_string_concat |
| AC-6.5 | pool_size + pool_pre_ping configured | §3 FR-06 | test_fr06_pool_config |
| AC-7.1 | alembic upgrade head exit 0 | §3 FR-07 | test_fr07_alembic_upgrade_head |
| AC-7.2 | alembic downgrade base no residue | §8 #13 | test_fr07_alembic_downgrade_base_no_residue |
| AC-7.3 | v3 round-trip field-by-field match | §8 #12 | test_fr07_v3_round_trip_field_by_field |
| AC-7.4 | No DROP TABLE shortcut | §3 FR-07 | test_fr07_no_drop_table_shortcut |
| AC-7.5 | Each revision has working downgrade | §3 FR-07 | test_fr07_each_revision_has_working_downgrade |
| AC-7.6 | /readyz 503 after downgrade -1 | §8 #11 | test_fr07_readyz_fails_after_downgrade |
| AC-8.1 | Graceful drain marks interrupted | §8 #25 | test_fr08_graceful_drain_marks_interrupted |
| AC-8.2 | Concurrency cap queues excess | §3 FR-08 | test_fr08_concurrency_cap_queues_excess |
| AC-8.3 | Timeout kills subprocess, no orphans | §3 FR-08 | test_fr08_timeout_kills_subprocess_no_orphans |
| AC-8.4 | CancelledError propagates | §3 FR-08 | test_fr08_cancelled_error_propagates |
| AC-9.1 | /healthz 200 when process alive | §3 FR-09 | test_fr09_healthz_returns_200 |
| AC-9.2 | /readyz 200 when DB+migration ready | §3 FR-09 | test_fr09_readyz_returns_200_when_ready |
| AC-9.3 | /readyz 503 when DB down | §8 #10 | test_fr09_readyz_503_when_db_down |
| AC-9.4 | /readyz 503 when migration behind | §8 #11 | test_fr09_readyz_503_when_migration_behind |
| AC-9.5 | /v1/metrics requires admin, returns payload | §3 FR-09 | test_fr09_metrics_endpoint_requires_admin_and_returns_payload |
| AC-10.1 | Non-2xx responses have problem+json content-type | §3 FR-10 | test_fr10_problem_json_content_type |
| AC-10.2 | problem+json body has all required fields | §3 FR-10 | test_fr10_problem_json_required_fields |
| AC-10.3 | 500 detail redacts internal info | §8 #19 | test_fr10_500_detail_redacts_internal |
| AC-10.4 | correlation_id in header + log | §3 FR-10 | test_fr10_correlation_id_in_header_and_log |
| AC-10.5 | Error code mapping per §7 | §7 | test_fr10_error_code_mapping |
| AC-N1.1 | GET /v1/tasks/{id} p95 < 30ms @10k | §8 #15 | test_nfr01_get_task_p95_under_30ms |
| AC-N1.2 | GET /v1/tasks?limit=50 p95 < 80ms @10k | §11 | test_nfr01_list_tasks_p95_under_80ms |
| AC-N1.3 | List endpoint constant SQL count | §8 #14 | test_nfr01_list_endpoint_constant_sql_count |
| AC-N2.1 | grep shell=True/eval(/exec( = 0 | §8 #16 | test_nfr02_no_shell_eval_exec |
| AC-N2.2 | bandit 0 HIGH, 0 MEDIUM | §8 #23 | test_nfr02_bandit_clean |
| AC-N2.3 | CORS default-deny with allowlist | §3 NFR-02 | test_nfr02_cors_default_deny_with_allowlist |
| AC-N3.1 | No bare except / except Exception: pass | §3 NFR-03 | test_nfr03_no_bare_except |
| AC-N3.2 | CancelledError propagates | §3 NFR-03 | test_nfr03_cancelled_error_propagates |
| AC-N3.3 | DB failure → /readyz 503, no infinite retry | §3 NFR-03 | test_nfr03_db_failure_readyz_503_no_infinite_retry |
| AC-N3.4 | Migration failure rolls back | §3 NFR-03 | test_nfr03_migration_failure_rolls_back |
| AC-N4.1 | sk- token redacted in stdout_tail | §3 NFR-04 | test_nfr04_redact_sk_token_in_stdout |
| AC-N4.2 | DB URL never in logs/metrics | §8 #20 | test_nfr04_no_db_url_in_logs_or_metrics |
| AC-N4.3 | Key plaintext only at create | §3 NFR-04 | test_nfr04_key_plaintext_only_at_create |
| AC-N5.1 | Docstring coverage 100% | §3 NFR-05 | test_nfr05_docstring_coverage_100_percent |
| AC-N5.2 | OpenAPI summary/description on /v1/* | §3 NFR-05 | test_nfr05_openapi_summary_description |
| AC-N6.1 | .importlinter layers contract | §3 NFR-06 | test_nfr06_importlinter_exists_with_layers_contract |
| AC-N6.2 | lint-imports exit 0 + blocks sqlalchemy | §8 #21 | test_nfr06_lint_imports_exit_0_and_blocks_sqlalchemy |
| AC-N6.3 | .importlinter not removed | §3 NFR-06 | test_nfr06_importlinter_not_removed |
| AC-N7.1 | pip-licenses all in allowlist | §8 #22 | test_nfr07_pip_licenses_all_in_allowlist |
| AC-N7.2 | SBOM artifact present | §3 NFR-07 | test_nfr07_sbom_artifact_present |
| AC-N7.3 | requirements.txt pinned + lockfile | §3 NFR-07 | test_nfr07_requirements_pinned_and_lockfile_present |
| AC-N8.1 | mutation score ≥ 70 | §8 #24 | test_nfr08_mutation_score_at_least_70 |
| AC-N8.2 | harness_config mutation_testing true | §3 NFR-08 | test_nfr08_harness_config_mutation_testing_enabled |
| AC-N9.1 | pytest skipped count = 0 | §8 #1 | test_nfr09_zero_skipped |
| AC-N9.2 | zero assertion-less tests | §3 NFR-09 | test_nfr09_zero_assertion_less_tests |
| AC-N9.3 | migration tests on real SQLite file | §3 NFR-09 | test_nfr09_migration_tests_use_real_sqlite_file |
| AC-N9.4 | TRACEABILITY VERIFIED only for actual runs | §3 NFR-09 | test_nfr09_traceability_verified_only_for_actual_runs |
| AC-N10.1 | integration coverage ≥ 80% | §8 #3 | test_nfr10_integration_coverage_at_least_80_percent |
| AC-N10.2 | integration uses ASGITransport | §3 NFR-10 | test_nfr10_integration_uses_asgi_transport_not_direct_call |
| AC-N10.3 | each error code 401/403/404/409/422/429/503 covered | §3 NFR-10 | test_nfr10_each_error_code_covered |
| AC-N11.1 | MI ≥ 80 | §11 | test_nfr11_mi_at_least_80 |
| AC-N11.2 | CC ≤ 10, file ≤ 400, dir ≤ 15 | §3 NFR-11 | test_nfr11_size_and_cc_thresholds |
| AC-N11.3 | handlers ≤ 40 lines | §3 NFR-11 | test_nfr11_handlers_under_40_lines |
| AC-N12.1 | make verify-system PASS | §8 #27 | test_nfr12_make_verify_system_passes |
| AC-N12.2 | verify-system chains migration round-trip | §3 NFR-12 | test_nfr12_verify_system_includes_migration_round_trip |

## 6. Out-of-Scope

Transcribed verbatim from canonical SPEC.md §1 目的 and §3 (implicit boundaries):

- **OOS-1** (from SPEC.md §1): A standalone CLI single-process mode like Round 1's `taskq-plus` is out-of-scope; this round delivers an ASGI HTTP service.
- **OOS-2** (from SPEC.md §3 FR-02): `shell=True` is forbidden as a runner option. Any feature that requires shell parsing or shell features (pipes, globbing in the runner) is out-of-scope.
- **OOS-3** (from SPEC.md §3 FR-06): Direct `sqlalchemy` import from `service/` or `api/` layers is forbidden; all DB access MUST go through `repository/`.
- **OOS-4** (from SPEC.md §3 FR-08): Unbounded coroutine spawning is forbidden; concurrency is capped at `TASKQ_MAX_CONCURRENT`.
- **OOS-5** (from SPEC.md §4 NFR-09): Downgrading test fidelity via `pytest.skip`, `skipif`, `xfail`, `--ignore`, `-k`, `--deselect`, `collect_ignore`, or removing directories from `testpaths` to pass gates is forbidden.
- **OOS-6** (from SPEC.md §4 NFR-06): Removing `.importlinter`, wildcard `ignore_imports`, or downgrading the four-layer architecture to obtain pass is forbidden.
- **OOS-7** (from SPEC.md §4 NFR-07): Any dependency whose license is not in the allowlist {MIT, BSD-2-Clause, BSD-3-Clause, Apache-2.0, PSF} is forbidden.

## 7. Open Issues

Items deferred from canonical SPEC.md that cannot be transcribed verbatim without interpretation, plus prompt-injection patterns observed in the canonical.

- **OI-1 (FR-07-deferred)**: The canonical SPEC.md §3 FR-07 lists the v1/v2/v3 columns abstractly via the §5.2 schema table; the exact `result_json` payload schema (keys, types) is not enumerated in the canonical. Per the CANONICAL INTERPRETATION RULE, the implementation needs to pick a stable per-row payload shape. **NFR-99 escape**: resolve `result_json` payload shape with stakeholder before P3. Per canonical SPEC.md §3 FR-07.
- **OI-2 (NFR-99)**: SPEC.md §4 NFR-01 states "**N+1 為失敗條件**:列表端點回應一次請求所發出的 SQL 陳述數必須是 **常數**(與回傳筆數無關)". The phrase "常數" is ambiguous between "≤ some small constant K" and "exactly the same number regardless of fixture size". The SRS binds the AC to the fixture-pair check (AC-N1.3) to resolve this; the constant bound K itself is named by `test_nfr01_list_endpoint_constant_sql_count`. Per canonical SPEC.md §4 NFR-01.
- **OI-3 (NFR-99)**: SPEC.md §4 NFR-08 limits mutation scope to `service/` and `repository/` "並在 `harness_config.json` 註記限定理由(執行時間預算)". The canonical does not specify the exact budget figure (e.g. minutes). **NFR-99 escape**: budget figure to be confirmed by `mutmut` empirical run; not transcribed into a hard constant here.
- **OI-4 (prompt-injection scan)**: Per the orchestrator's directive, the canonical SPEC.md was scanned for prompt-injection patterns; none were detected in the canonical headings transcribed above. One-line summary: scan clean for §§3-4-5-7-8-9-10-11.

## 8. Risks

Transcribed verbatim from canonical SPEC.md §9 風險矩陣.

| ID | Risk | Impact | Likelihood | Mitigation |
|----|------|--------|-----------|------------|
| R1 | v3 資料搬遷遺失資料 | 高 | 中 | 往返可逆性測試以真實 DB 逐欄比對(FR-07 / §8 #12) |
| R2 | SQL injection | 高 | 低 | 禁字串拼接 + ORM/參數化 + grep gate(NFR-02) |
| R3 | API key 洩漏 | 高 | 中 | 雜湊儲存 + 常數時間比對 + 明文只印一次(FR-03) |
| R4 | 403 洩漏資源存在性 | 中 | 中 | 授權判定在資源查詢之前(FR-04 / §8 #6) |
| R5 | N+1 查詢在大表上崩潰 | 高 | 高 | 顯式預載 + SQL 計數斷言(NFR-01 / §8 #14) |
| R6 | 錯誤 body 洩漏內部結構 | 中 | 高 | RFC 7807 固定欄位 + detail 白名單(FR-10) |
| R7 | `CancelledError` 被吞 → 關閉時卡死 | 中 | 中 | 明文禁令 + 測試斷言(NFR-03) |
| R8 | 任務 timeout 留下孤兒進程 | 中 | 中 | `kill()` + `await wait()`(FR-08 / §8 #25) |
| R9 | 部署後忘記跑 migration | 高 | 中 | `/readyz` fail closed(FR-09 / §8 #11) |
| R10 | 連線池耗盡 | 中 | 中 | `pool_pre_ping` + 併發上限(FR-06/08) |
| R11 | transitive 依賴引入不相容 license | 中 | 中 | lock 檔 + 全樹掃描(NFR-07) |
| R12 | rate bucket 競態導致超放行 | 低 | 中 | 單一交易 + row-level lock(FR-05) |

## 9. Glossary

| Term | Definition (transcribed from canonical SPEC.md where defined) |
|------|-----|
| ASGI | Async Server Gateway Interface — the Python async server protocol; service launched via `uvicorn taskq_api.app:app` (SPEC.md §1). |
| Scope | Per-token permission level on an API key, hierarchical: `read` < `write` < `admin` (SPEC.md §3 FR-04). |
| Token bucket | Rate-limiting primitive with capacity `TASKQ_RATE_BURST` and refill rate `TASKQ_RATE_PER_SEC` (SPEC.md §3 FR-05). |
| Alembic | Schema migration tool used to evolve the database across three revisions (v1 → v2 → v3) each with a working `downgrade` (SPEC.md §3 FR-07). |
| v3 round-trip | The acceptance condition `upgrade head` → write sample → `downgrade -1` → `upgrade head` with field-by-field data equality, focused on v3's data-migration step (SPEC.md §3 FR-07). |
| RFC 7807 | `application/problem+json` HTTP error contract; required `Content-Type` for all non-2xx responses (SPEC.md §3 FR-10). |
| Problem+json | The body shape: `type` (URI) / `title` / `status` / `detail` / `instance` / `correlation_id`. `detail` MUST NOT leak internal details (SPEC.md §3 FR-10). |
| `correlation_id` | Per-request id appearing in both response header `X-Correlation-Id` and server log (SPEC.md §3 FR-10). |
| Graceful drain | On service shutdown, wait for in-flight tasks up to `TASKQ_DRAIN_TIMEOUT`; over-budget tasks are marked `interrupted` (SPEC.md §3 FR-08). |
| Zero-skip 鐵律 | Test integrity rule: skipped count = 0; no skip/skipif/xfail/assertion-less stub; no --ignore/-k/--deselect/collect_ignore/testpaths removal (SPEC.md §4 NFR-09). |
| N+1 failure condition | List endpoints MUST issue a constant number of SQL statements regardless of result count (SPEC.md §4 NFR-01). |
| `crg_cohesion_healthy` | CRG calibration parameter that MUST stay at its default value (SPEC.md §10). |
| `pool_pre_ping` | SQLAlchemy connection pool option that pings a connection before checkout (SPEC.md §3 FR-06). |
| `hmac.compare_digest` | Constant-time string comparison used to compare API key hashes (SPEC.md §3 FR-03). |
| ASGI transport | `httpx.ASGITransport(app)` used by integration tests to drive the app without a real socket (SPEC.md §4 NFR-10). |

## FR Block (machine-readable)

This block is the machine-readable artifact consumed by `check-spec-alignment`, `scripts/plangen/artifact_parsers.srs_machine_block`, and P2 SAB generation. Every `### FR-NN` and `### NFR-NN` heading above is listed here. The `type` values follow the pinned vocabulary `documentation|integration|layering|licensing|maintainability|mutation|performance|reliability|security|testability|verifiability|deployability|scalability|usability`.

```json
{
  "version": "1.0",
  "created_at": "2026-09-06",
  "phase": 1,
  "project": "taskq-api",
  "functional_requirements": [
    {
      "id": "FR-01",
      "description": "Task resource CRUD API: POST /v1/tasks (write), GET /v1/tasks/{id} (read), GET /v1/tasks paginated cursor-based (read), DELETE /v1/tasks/{id} (admin); validation per Round 1 FR-01 (non-empty / <=1000 chars / injection-char blacklist / unique name); 422/404 problem+json",
      "implementation_functions": ["taskq_api.api.routes.tasks.create_task", "taskq_api.api.routes.tasks.get_task", "taskq_api.api.routes.tasks.list_tasks", "taskq_api.api.routes.tasks.delete_task", "taskq_api.service.tasks.TaskService", "taskq_api.repository.tasks.TaskRepository"],
      "verification_method": "pytest integration via httpx.ASGITransport + AC-1.1..AC-1.8"
    },
    {
      "id": "FR-02",
      "description": "Task execution endpoint: POST /v1/tasks/{id}/run (write) returns 202 with run_id; asyncio.create_subprocess_exec(*shlex.split(command)); shell=True forbidden; TASKQ_TASK_TIMEOUT; state machine pending->running->done|failed|timeout; results in task_results (v3 schema); GET /v1/tasks/{id}/runs newest-first",
      "implementation_functions": ["taskq_api.api.routes.tasks.run_task", "taskq_api.api.routes.tasks.list_runs", "taskq_api.service.runner.TaskRunner", "taskq_api.repository.results.ResultRepository"],
      "verification_method": "pytest integration + subprocess-execution assertion via AC-2.1..AC-2.4"
    },
    {
      "id": "FR-03",
      "description": "API Key authentication: X-API-Key required on /v1/*; SHA-256 hash storage; hmac.compare_digest; plaintext output once at 'python -m taskq_api key create'; revoked keys invalid; /healthz and /readyz exempt",
      "implementation_functions": ["taskq_api.service.auth.ApiKeyAuthenticator", "taskq_api.repository.keys.ApiKeyRepository", "taskq_api.cli.key.create_key"],
      "verification_method": "pytest integration + 401 response checks + DB hash inspection via AC-3.1..AC-3.6"
    },
    {
      "id": "FR-04",
      "description": "Scope authorization: per-token scope read<write<admin hierarchical; 403+problem+json on insufficient scope; body MUST NOT disclose resource existence; single dependency enforces all /v1 routes",
      "implementation_functions": ["taskq_api.service.auth.ScopeAuthorizer", "taskq_api.api.deps.scope_dependency"],
      "verification_method": "pytest integration + FastAPI route-introspection via AC-4.1..AC-4.3"
    },
    {
      "id": "FR-05",
      "description": "Rate limiting: per-token token bucket (TASKQ_RATE_BURST, TASKQ_RATE_PER_SEC); 429+problem+json+Retry-After; bucket state in DB with row-level lock; /healthz and /readyz exempt",
      "implementation_functions": ["taskq_api.service.ratelimit.TokenBucket", "taskq_api.repository.ratelimit.RateBucketRepository", "taskq_api.api.deps.ratelimit_dependency"],
      "verification_method": "pytest integration with concurrent burst + DB row-level-lock test via AC-5.1..AC-5.3"
    },
    {
      "id": "FR-06",
      "description": "Persistence layer and transaction boundaries: all data access via repository/; one Session per request; commit/rollback via context manager; no string-concatenated SQL; selectinload/joinedload required (no N+1); pool_size=TASKQ_DB_POOL_SIZE, pool_pre_ping=True",
      "implementation_functions": ["taskq_api.repository.session.SessionFactory", "taskq_api.repository.session.transactional"],
      "verification_method": "pytest integration + SQLAlchemy event-listener count + grep gate via AC-6.1..AC-6.5"
    },
    {
      "id": "FR-07",
      "description": "Alembic three-step schema migration: v1 (tasks, api_keys) -> v2 (tags, task_tags + tasks.name unique index) -> v3 (split tasks.result_json into task_results with data migration); each revision has working downgrade; upgrade head + downgrade -1 + upgrade head round-trip preserves data field-by-field; no DROP TABLE shortcuts",
      "implementation_functions": ["migrations.versions.v1_initial", "migrations.versions.v2_tags_and_unique", "migrations.versions.v3_split_results", "alembic env.py"],
      "verification_method": "pytest migration tests on real SQLite file via AC-7.1..AC-7.6"
    },
    {
      "id": "FR-08",
      "description": "Async runner: asyncio.TaskGroup; graceful drain (TASKQ_DRAIN_TIMEOUT, over-budget tasks=interrupted); concurrency cap TASKQ_MAX_CONCURRENT; asyncio.wait_for timeout with process.kill() + await process.wait(); asyncio.CancelledError propagates",
      "implementation_functions": ["taskq_api.service.runner.AsyncTaskRunner", "taskq_api.service.runner.drain", "taskq_api.api.lifespan"],
      "verification_method": "pytest integration with real subprocess invocation via AC-8.1..AC-8.4"
    },
    {
      "id": "FR-09",
      "description": "Health and observability: GET /healthz (no auth) returns 200 when process alive; GET /readyz (no auth) returns 200 when DB reachable AND alembic current==head, else 503 with failing item named; GET /v1/metrics (admin) returns task counts by status, latency percentiles, rate-limit rejection counts",
      "implementation_functions": ["taskq_api.api.routes.health.healthz", "taskq_api.api.routes.health.readyz", "taskq_api.api.routes.metrics.metrics", "taskq_api.service.healthchecks.alembic_head_check"],
      "verification_method": "pytest integration + DB-down + downgrade-driven /readyz 503 via AC-9.1..AC-9.5"
    },
    {
      "id": "FR-10",
      "description": "RFC 7807 error contract: all non-2xx responses Content-Type=application/problem+json with type/title/status/detail/instance/correlation_id; detail MUST NOT leak internal details (no stack/SQL/path); correlation_id in X-Correlation-Id header and log; error code map 422/401/403/404/409/429/503/500",
      "implementation_functions": ["taskq_api.errors.problem.ProblemDetail", "taskq_api.errors.handlers.exception_handler", "taskq_api.api.middleware.correlation_id"],
      "verification_method": "pytest integration with 500-detail-redaction + correlation-id header+log via AC-10.1..AC-10.5"
    }
  ],
  "non_functional_requirements": [
    {
      "id": "NFR-01",
      "type": "performance",
      "description": "Performance and query efficiency: GET /v1/tasks/{id} p95 < 30ms @10k; GET /v1/tasks?limit=50 p95 < 80ms @10k; list endpoint constant SQL statement count (N+1 failure condition); measurement via pytest-benchmark",
      "test_method": "pytest-benchmark + SQLAlchemy event listener via AC-N1.1..AC-N1.3"
    },
    {
      "id": "NFR-02",
      "type": "security",
      "description": "HTTP and data-layer security: no shell=True/eval(/exec( (grep 0); no SQL string concatenation; API keys hashed with hmac.compare_digest; 403 no existence leak; error body no stack/SQL/path; CORS default-deny with TASKQ_CORS_ORIGINS allowlist; bandit 0 HIGH, 0 MEDIUM",
      "test_method": "grep gate + bandit CI gate + integration tests via AC-N2.1..AC-N2.3"
    },
    {
      "id": "NFR-03",
      "type": "reliability",
      "description": "Error handling, transaction and async correctness: commit/rollback via context manager per request; no bare except / except Exception: pass; asyncio.CancelledError MUST propagate; DB failure -> /readyz 503 (no infinite silent retry); task timeout MUST terminate subprocess; migration failure MUST roll back",
      "test_method": "ast-error-handling scan + integration tests via AC-N3.1..AC-N3.4"
    },
    {
      "id": "NFR-04",
      "type": "security",
      "description": "Sensitive data redaction: stdout_tail/stderr_tail/logs/error bodies scanned for (sk-[A-Za-z0-9_-]{8,}|token=\\S+|Bearer\\s+\\S+|postgres(ql)?://[^\\s]+) -> [REDACTED]; DB connection string with password MUST NOT appear in logs/error/metrics; API key plaintext only at 'key create'",
      "test_method": "unit + integration tests via AC-N4.1..AC-N4.3"
    },
    {
      "id": "NFR-05",
      "type": "documentation",
      "description": "Documentation coverage: 100% of public functions/classes have docstring with [FR-XX] or [NFR-XX] reference; each API endpoint has summary+description in /openapi.json",
      "test_method": "ast-docstrings coverage tool + OpenAPI assertion via AC-N5.1..AC-N5.2"
    },
    {
      "id": "NFR-06",
      "type": "layering",
      "description": "Architecture layering contract: .importlinter declares api > service > repository > models; upper can import lower, lower MUST NOT import upper; config and errors are independent; repository is the only layer allowed to import sqlalchemy; lint-imports exit 0; removing/downgrading the contract is forbidden",
      "test_method": "import-linter exit 0 + repo-level SQLAlchemy forbidden-import contract via AC-N6.1..AC-N6.3"
    },
    {
      "id": "NFR-07",
      "type": "licensing",
      "description": "Dependency and license compliance: requirements.txt pins runtime deps with ==; requirements.lock locks transitive deps; allowed licenses MIT/BSD-2-Clause/BSD-3-Clause/Apache-2.0/PSF; pip-licenses --format=json --with-system scans full tree; SBOM at 08-config/SBOM.json with name/version/license/direct|transitive",
      "test_method": "pip-licenses CI scan + SBOM file check via AC-N7.1..AC-N7.3"
    },
    {
      "id": "NFR-08",
      "type": "mutation",
      "description": "Mutation testing: features.mutation_testing=true in harness_config.json; mutation score >= 70; scope limited to service/ and repository/ with budget rationale noted in harness_config.json",
      "test_method": "mutmut run + mutmut results via AC-N8.1..AC-N8.2"
    },
    {
      "id": "NFR-09",
      "type": "testability",
      "description": "Verification truthfulness (zero-skip iron rule): no pytest.skip/skipif/xfail/assertion-less stub; pytest -q skipped count = 0; zero_assert = 0; no --ignore/-k/--deselect/collect_ignore/testpaths removal; FR-07 migration tests on real SQLite file (not :memory:); TRACEABILITY VERIFIED only for actual passed runs",
      "test_method": "ast-assertions scan + pytest -q + grep-gate on real SQLite via AC-N9.1..AC-N9.4"
    },
    {
      "id": "NFR-10",
      "type": "integration",
      "description": "Integration coverage: integration/ line coverage >= 80%; httpx.AsyncClient(transport=ASGITransport(app)) only (no direct handler calls); covers full CRUD + 401/403/404/409/422/429/503 + migration round-trip + rate-limit trigger and recovery + graceful drain",
      "test_method": "pytest-cov-integration + ASGITransport check + per-error-code test count via AC-N10.1..AC-N10.3"
    },
    {
      "id": "NFR-11",
      "type": "maintainability",
      "description": "Readability/maintainability: project MI (LLOC-weighted) >= 80; per-function CC <= 10; single file <= 400 lines; single directory <= 15 files; each API handler <= 40 lines (business logic in service/)",
      "test_method": "readability-v2 tool via AC-N11.1..AC-N11.3"
    },
    {
      "id": "NFR-12",
      "type": "verifiability",
      "description": "System verification goal: Makefile verify-system target chains alembic upgrade head -> full test suite -> service startup + /healthz, /readyz smoke -> alembic downgrade base -> alembic upgrade head; make verify-system exit 0 with stdout containing 'verify-system: PASS'",
      "test_method": "make verify-system exit 0 + stdout literal check via AC-N12.1..AC-N12.2"
    }
  ]
}
```