# Architecture Decision Records (ADR) — taskq-api

> Collection of architectural decisions for `taskq-api`. Each `## ADR-NNN`
> block captures one decision with context, choice, alternatives, and
> consequences. Source of truth: `02-architecture/SAD.md` (binding SAB
> contract). Phase-2 orchestrator loads this file via
> `diskPrefix: '# Architecture Decision Records'`.

## ADR-001: Python 3.11 stdlib-first runtime

### Status
Accepted

### Satisfies
FR-08 requirement (SRS §3 subprocess-drain specification); NFR-07 SBOM
requirement (SRS §6 license-compliance specification).

### Context
`taskq-api` is delivered as a single-process ASGI service. The deployment
target must run reproducibly in CI, in `make verify-system`, and inside a
`.venv` whose Python interpreter is `.venv/bin/python`. We need a runtime
that supports the async ergonomics required by FR-08 (subprocess drain),
structural pattern matching for repository parsing, and `tomllib` for
reading `.methodology/harness_config.json` without a third-party parser.

### Decision
- Pin interpreter: **Python 3.11.15** (verified via `.venv/bin/python --version`).
- Prefer **standard-library** solutions for redaction, hashing, settings IO,
  and redaction regex (`hashlib`, `hmac`, `secrets`, `logging`, `re`,
  `dataclasses`, `tomllib`).
- Add only libraries whose removal would force rewriting business logic:
  FastAPI, SQLAlchemy 2.x, Alembic, Pydantic, uvicorn, httpx (test).

### Alternatives considered
- **Python 3.12+**: rejected — `TaskGroup` backport not needed (3.11 has it),
  and 3.11 LTS is what the harness `.venv` ships.
- **Go/Rust runtime**: rejected — NFR-07 (license allowlist) and FR-08
  asyncio subprocess semantics map cleanly to CPython.

### Consequences
- Positive: smallest SBOM (NFR-07); predictable `make verify-system`.
- Negative: typing is runtime-enforced only via Pydantic; pure domain code
  needs explicit guards to avoid `Any` leakage.

## ADR-002: Five-layer architecture with two independence modules

### Status
Accepted

### Satisfies
NFR-06 architecture requirement (SRS §6 layering specification).

### Context
SAD §1 fixes a layering contract `api > service > repository > models`
because NFR-06 forbids `sqlalchemy` imports outside `repository/` and
`models/`. Two modules — `config.py` and `errors.py` — must be importable
from every layer without creating upward edges. The layering maps 1:1 to
CRG communities for impact-radius review.

### Decision
- Five layers: `api`, `service`, `repository`, `models`, `migrations`.
- Two independence modules: `taskq_api.config` (Pydantic settings) and
  `taskq_api.errors` (RFC 7807 `ProblemDetail`/`ProblemException`).
- Enforced by **import-linter** (`.importlinter`) — forbidden contracts
  block `sqlalchemy` imports and the four cross-layer edges not listed in
  `allowed_dependencies`.

### Alternatives considered
- **Flat structure (no layers)**: rejected — NFR-06 + mutation testing
  (NFR-08) become unscoreable.
- **Hexagonal/ports-and-adapters**: rejected — overkill for a 5-CR-file
  service; would violate Simplicity First without buying testability.

### Consequences
- Positive: high cohesion per CRG community; mutation score (NFR-08) is
  measurable per layer.
- Negative: any cross-cutting concern (e.g. correlation_id) must be passed
  through `APIContext` rather than via ambient state.

## ADR-003: ASGI via FastAPI + uvicorn

### Status
Accepted

### Satisfies
FR-01..FR-10 REST requirement (SRS §3 API specification); NFR-05 OpenAPI
documentation requirement (SRS §6 documentation specification).

### Context
FR-01..FR-10 require request validation, dependency injection for
auth/scope/rate-limit, RFC 7807 exception mapping, and OpenAPI emission
(NFR-05). A synchronous WSGI stack would block the event loop that FR-08
needs for concurrent subprocess execution.

### Decision
- **FastAPI** for routing, pydantic request/response models, and
  `Depends(...)` wiring for `require_auth` / `require_scope` /
  `consume_rate_token`.
- **uvicorn** as the ASGI server; `python -m taskq_api` boots the factory
  via `taskq_api.app:app`.
- Lifespan hook calls `runner.drain(timeout=settings.DRAIN_TIMEOUT)` on
  shutdown (FR-08) and registers CORS middleware.

### Alternatives considered
- **Starlette bare**: rejected — would require re-implementing pydantic
  binding and exception mapping that FastAPI already supplies.
- **gRPC**: rejected — SPEC §3 mandates REST + RFC 7807.

### Consequences
- Positive: native pydantic v2 + `Annotated` ergonomics; trivial OpenAPI.
- Negative: dependency on FastAPI's exception machinery — must register
  `ProblemException` handler before app starts serving.

## ADR-004: SQLAlchemy 2.x ORM with `session_scope` context manager

### Status
Accepted

### Satisfies
FR-06 transaction-boundary requirement (SRS §3 FR-06 specification);
NFR-01 performance requirement (SRS §6 NFR-01 specification); NFR-02
SQL-injection-prevention requirement (SRS §6 NFR-02 specification); NFR-03
error-handling requirement (SRS §6 NFR-03 specification).

### Context
FR-06 requires a transaction boundary per request. NFR-03 forbids bare
`except:` / `except Exception: pass` and mandates commit/rollback
discipline. NFR-02 forbids string-concatenated SQL. The repository layer
must therefore own all DB access via a single, well-typed context manager
that wraps commit/rollback.

### Decision
- Use **SQLAlchemy 2.x** with the new typed declarative API.
- Single boundary primitive: `session_scope()` context manager exported
  from `taskq_api.repository.session`. Commit on success, rollback on
  exception; raises whatever the service raised after rollback.
- Every `*_repository` function takes a `Session` argument; the service
  layer never holds a `Session` long-term.
- Rate-bucket writes use **`SELECT … FOR UPDATE`** (Postgres) or SQLite
  `BEGIN IMMEDIATE` to keep token-bucket updates atomic (FR-05, R12).
- Cursor pagination (`?cursor=`) on list endpoints to avoid OFFSET cost.

### Alternatives considered
- **Raw `sqlite3` / `psycopg`**: rejected — NFR-01 requires constant SQL
  count for list endpoints, which ORM-level `selectinload` provides.
- **Async SQLAlchemy (`AsyncSession`)**: rejected — subprocess execution
  (FR-08) blocks the loop anyway; sync session inside `run_in_threadpool`
  is simpler and matches the rest of the codebase.
- **Unit-of-Work pattern (hand-rolled)**: rejected — duplicates ORM
  identity-map semantics without benefit.

### Consequences
- Positive: linear commit/rollback path; mutation-test friendly (NFR-08).
- Negative: every request pays the cost of opening a session; acceptable
  given p95 budget in NFR-01.

## ADR-005: Alembic-managed schema evolution with reversible v3 split

### Status
Accepted

### Satisfies
FR-07 migration requirement (SRS §3 FR-07 specification); NFR-09
test-assertion requirement (SRS §6 NFR-09 specification); NFR-12
verification requirement (SRS §6 NFR-12 specification).

### Context
FR-07 requires three migrations (`v1_initial`, `v2_tags_and_unique_name`,
`v3_split_results`) where **v3 splits `tasks.result_json` into a dedicated
`task_results` table**. Each revision must be reversible (`downgrade()`
must work). NFR-09 forbids skip/xfail, so v3 round-trip must be exercised
against a real SQLite file in CI.

### Decision
- Three revisions under `migrations/versions/`:
  - `v1_initial` — `tasks`, `api_keys`.
  - `v2_tags_and_unique_name` — `tags`, `task_tags`, unique index on
    `tasks.name`.
  - `v3_split_results` — `task_results`; backfill from `tasks.result_json`
    in deterministic order; then `ALTER TABLE tasks DROP COLUMN
    result_json`. `downgrade()` re-merges `task_results` back into the
    column, then drops the table.
- `downgrade()` is implemented for every revision (no `op.execute("DROP
  TABLE …")` shortcuts that erase history).
- `make verify-system` runs `alembic upgrade head` then `alembic downgrade
  base` then `alembic upgrade head` to prove round-trip data preservation.

### Alternatives considered
- **Single-shot `Base.metadata.create_all`**: rejected — destroys the
  FR-07 audit trail and the round-trip property required by NFR-12.
- **Online migration tool (e.g. expand-contract by hand)**: rejected —
  SQLite/Postgres delta is small enough that a single in-place backfill is
  correct and testable.

### Consequences
- Positive: schema evolution is auditable; downgrade works.
- Negative: v3 is high-risk — listed in `high_risk_modules` and subject to
  mutation testing (NFR-08).

## ADR-006: Async subprocess execution via `asyncio.create_subprocess_exec`

### Status
Accepted

### Satisfies
FR-02 runner requirement (SRS §3 FR-02 specification); FR-08 subprocess
requirement (SRS §3 FR-08 specification); NFR-04 secret-leak prevention
requirement (SRS §6 NFR-04 specification).

### Context
FR-02/FR-08 require running a user-supplied `command` per task, with a
timeout, capturing `stdout` / `stderr` tails, and persisting the result.
Shell metacharacter injection is a security boundary (SAD §6 T-05).
Orphan processes on timeout are a reliability boundary (T-06).

### Decision
- Spawn via **`asyncio.create_subprocess_exec(*shlex.split(command),
  shell=False)`** — `shell=True` is forbidden by grep gate.
- Bound concurrency with `asyncio.Semaphore(settings.TASKQ_MAX_CONCURRENT)`
  inside an `asyncio.TaskGroup`.
- Timeout: `asyncio.wait_for(...)`; on expiry call `process.kill()` then
  `await process.wait()` to guarantee the child is reaped.
- On shutdown, the FastAPI lifespan awaits `runner.drain(timeout=...)` to
  finish in-flight runs.
- `service.redaction.scrub(...)` runs over `stdout`/`stderr` tails before
  persistence and before `/v1/metrics` emission (NFR-04).

### Alternatives considered
- **`ThreadPoolExecutor` + `subprocess.run`**: rejected — drains the kernel
  scheduler and complicates timeout/cancel semantics.
- **`shell=True` with quote escaping**: rejected — T-05 mitigation; a
  single missed quote is RCE.
- **`os.system`**: rejected — equivalent to `shell=True` plus no PID
  return for kill.

### Consequences
- Positive: shell injection blocked; orphans prevented; bounded
  parallelism.
- Negative: `shlex.split` will mis-tokenise shell-only syntax (`>`, `|`,
  `&&`) — by design; users supply argv-style commands.

## ADR-007: API-key authentication (SHA-256 + constant-time compare)

### Status
Accepted

### Satisfies
FR-03 authentication requirement (SRS §3 FR-03 specification); FR-04
scope requirement (SRS §3 FR-04 specification); NFR-02 security
requirement (SRS §6 NFR-02 specification).

### Context
FR-03/FR-04 require an `X-API-Key` header validated against a hashed
secret, with scope hierarchy `read < write < admin`. The auth check must
happen **before** resource fetch so a 403 body never reveals existence
(T-03). The hash function must allow constant-time comparison.

### Decision
- Store only the **SHA-256** of the API key in `api_keys.key_hash`.
- Lookup by raw key, hash on the fly, compare with
  `hmac.compare_digest`.
- A single `require_scope(level)` dependency enforces the hierarchy
  **before** the resource handler runs.
- Revocation: `api_keys.revoked_at IS NULL` enforced in the repository
  query; revoked keys return 401, not 403 (no existence leak).
- `bandit -r` 0 HIGH/MEDIUM gate (NFR-02).

### Alternatives considered
- **bcrypt/argon2**: rejected — service is stateless and bcrypt cost
  dominates the per-request budget; SHA-256 + 256-bit key space is
  sufficient for this deployment (SPEC §5).
- **JWT**: rejected — adds a parser dependency and a clock-skew NFR; out
  of scope per SPEC §3.
- **mTLS**: rejected — operationally heavier; SPEC does not require it.

### Consequences
- Positive: O(1) auth, constant-time compare, no DB-side bcrypt cost.
- Negative: SHA-256 of low-entropy keys is brute-forceable offline;
  accepted because the allowlist enforces ≥ 32-byte random keys.

## ADR-008: Token-bucket rate limiting with row-level lock

### Status
Accepted

### Satisfies
FR-05 rate-limit requirement (SRS §3 FR-05 specification).

### Context
FR-05 requires per-API-key rate limiting. T-07 documents the race:
two concurrent requests must not both succeed past the bucket cap. The
algorithm must be implementable inside the existing `session_scope`
transaction (FR-06).

### Decision
- Persist bucket state in `rate_buckets` (`key_id`, `tokens`, `last_refill`).
- One transaction per request: `SELECT … FOR UPDATE` (or SQLite
  `BEGIN IMMEDIATE`) the bucket row, refill by elapsed seconds, decrement,
  commit.
- 429 + `Retry-After` when `tokens < 1`; never apply rate limit to
  `/healthz` or `/readyz`.

### Alternatives considered
- **In-memory LRU + TTL**: rejected — multi-process deploy loses state;
  race window widens.
- **Sliding-window log**: rejected — O(N) storage per key vs O(1) for
  token bucket.

### Consequences
- Positive: atomic; matches existing `session_scope` discipline.
- Negative: write amplification (every request writes the bucket row);
  accepted given expected RPS.

## ADR-009: RFC 7807 error model via `errors.ProblemException`

### Status
Accepted

### Satisfies
FR-10 problem+json requirement (SRS §3 FR-10 specification).

### Context
FR-10 requires problem+json responses. T-09 forbids leaking stack traces,
SQL fragments, or file paths in the `detail` field. The error primitive
must be importable from any layer without violating NFR-06.

### Decision
- `taskq_api.errors.ProblemDetail` dataclass + `ProblemException(status,
  type, detail, **extensions)`.
- `api/exception_handlers.py` maps `ProblemException` → JSON body with
  `Content-Type: application/problem+json` and a `correlation_id` echoed
  in `X-Correlation-Id`.
- `detail` is a fixed whitelisted message; unhandled exceptions are mapped
  to a generic `"internal error"` detail; full traceback goes to the
  structured logger only.
- Service modules raise `ProblemException`; they do not return error
  tuples.

### Alternatives considered
- **Plain `HTTPException`**: rejected — no `type` field, no extensions,
  and inconsistent content-type.
- **Error envelopes (`{"error": ...}`)**: rejected — violates FR-10.

### Consequences
- Positive: machine-parseable errors; predictable content-type.
- Negative: every error site must import `errors`; easy to drift to
  `HTTPException` if linting is lax.

## ADR-010: Cursor-based pagination + eager-loaded relations

### Status
Accepted

### Satisfies
FR-01 list-endpoint requirement (SRS §3 FR-01 specification); NFR-01
performance requirement (SRS §6 NFR-01 specification).

### Context
NFR-01 requires constant SQL count for list endpoints regardless of
result size (R5). The default `select(Task).all()` plus lazy attribute
access triggers N+1. OFFSET pagination also degrades at depth.

### Decision
- List endpoints use opaque `?cursor=` cursors (encoded primary-key set
  + tiebreaker), never OFFSET.
- Relations pre-loaded with `selectinload(Task.tags)` / `joinedload`
  per endpoint; N+1 detection via SQLAlchemy event-listener test that
  asserts constant `select` count for `GET /v1/tasks` regardless of N.
- `pytest-benchmark` enforces p95 budgets in CI.

### Alternatives considered
- **OFFSET pagination**: rejected — degrades non-linearly; R5 violation.
- **Keyset pagination on a single column**: rejected — `tasks.id` is the
  only stable ordering; cursor encodes the id set.

### Consequences
- Positive: bounded SQL count; bounded latency.
- Negative: cursor format is opaque; clients cannot construct cursors by
  hand.

## ADR-011: Redaction layer (`service.redaction.scrub`)

### Status
Accepted

### Satisfies
NFR-04 secret-handling requirement (SRS §6 NFR-04 specification).

### Context
T-08 requires that stdout/stderr tails and metric lines never contain
secrets (`sk-…`, `token=…`, `Bearer …`, `postgres://…`). The DB URL
(`TASKQ_DB_URL`) must never appear in logs (SAD §2.3.2).

### Decision
- `service.redaction.scrub(text: str) -> str` replaces any line matching
  the redaction regex with `[REDACTED]`.
- Applied at three sinks: persistence in `run_repository.insert`,
  `/v1/metrics` emission, and the application logging filter.
- `correlation_id` never carries the request payload.

### Alternatives considered
- **Log-scrubbing only at handler**: rejected — secrets still land in
  `task_results.stdout_tail`.
- **Allowlist instead of denylist**: rejected — NFR-04 enumerates the
  must-scrub set; allowlist would need a per-instrument audit.

### Consequences
- Positive: secrets never leave the process unintentionally.
- Negative: regex is greedy; legitimate log lines that look like
  `token=…` (e.g. CI token strings) are redacted — accepted.

## ADR-012: CORS default-deny + credentials disabled

### Status
Accepted

### Satisfies
NFR-02 CORS requirement (SRS §6 NFR-02 specification).

### Context
T-10 requires that CORS misconfiguration not allow cross-origin
credentialed access. SPEC §5.1 provides `TASKQ_CORS_ORIGINS`.

### Decision
- `CORSMiddleware(allow_origins=settings.TASKQ_CORS_ORIGINS or [],
  allow_credentials=False, ...)`.
- Empty `TASKQ_CORS_ORIGINS` ⇒ reject all cross-origin requests
  (NFR-02, "CORS default reject").

### Alternatives considered
- **`allow_origins=["*"]`**: rejected — explicitly forbidden by T-10.
- **Per-route CORS opt-in**: rejected — uniform policy is simpler and
  matches the single origin-list configuration.

### Consequences
- Positive: closed by default; one env var to widen.
- Negative: SPA clients must set `TASKQ_CORS_ORIGINS` explicitly to work
  cross-origin.

## ADR-013: Lifespan-driven runner drain on shutdown

### Status
Accepted

### Satisfies
FR-08 shutdown requirement (SRS §3 FR-08 specification).

### Context
FR-08 requires that in-flight runs finish or are cancelled before the
process exits, with a bounded timeout (`TASKQ_DRAIN_TIMEOUT`). Without
this, an OOM or `kill -TERM` leaves orphan subprocesses holding stdout
pipes.

### Decision
- `app.py` lifespan `__aexit__` calls `runner.drain(timeout=...)`.
- `runner.drain` awaits every outstanding `asyncio.Task` in the
  `TaskGroup`; on timeout it `process.kill()` + `await process.wait()`
  for each.

### Alternatives considered
- **`SIGTERM` handler that re-execs**: rejected — loses in-flight request
  state; uvicorn already exposes lifespan.
- **No drain (let kernel reap)**: rejected — orphan pipe FDs leak; T-06.

### Consequences
- Positive: clean shutdown; no orphan processes.
- Negative: shutdown latency bounded by `TASKQ_DRAIN_TIMEOUT`; ops must
  tune it.

## ADR-014: `make verify-system` as the exit-gate verification target

### Status
Accepted

### Satisfies
NFR-12 verification-target requirement (SRS §6 NFR-12 specification).

### Context
NFR-12 mandates a single, repeatable verification command that exercises
the delivered entry point end-to-end. Gates 2/3/4 must pass this command.

### Decision
- `Makefile` target `verify-system` runs:
  1. `alembic upgrade head` against a real SQLite file.
  2. `pytest 03-development/tests -q` (skipped == 0 per NFR-09).
  3. `uvicorn taskq_api.app:app` + `/healthz` + `/readyz` smoke.
  4. `alembic downgrade base` then `alembic upgrade head` (v3 round-trip).
  5. `GET /v1/metrics` to force the metrics module to execute.
- Exits 0 and prints `verify-system: PASS` on success.

### Alternatives considered
- **`tox` matrix**: rejected — adds a runner dependency not in the
  NFR-07 allowlist for the deliverable.
- **Custom shell script**: rejected — `make` is already standard on
  Linux/macOS and matches the rest of the tooling.

### Consequences
- Positive: one command, one exit code, one log line.
- Negative: `make` must be installed in CI image; documented in README.

## ADR-015: import-linter as the layering enforcement gate

### Status
Accepted

### Satisfies
NFR-06 layering-enforcement requirement (SRS §6 NFR-06 specification).

### Context
NFR-06 requires machine-checked architecture constraints, not just
reviewer discipline. The forbidden contracts (`sqlalchemy` outside
`repository`/`models`, no circular deps) must fail CI when violated.

### Decision
- `.importlinter` declares four allowed edges:
  `api > service > repository > models`, plus `service > models` and
  `repository > models`.
- Forbidden contract: `sqlalchemy` must not be importable from any
  module outside `repository/` and `models/`.
- `lint-imports` runs in `make verify-system`; non-zero fails the gate.

### Alternatives considered
- **Manual review only**: rejected — drift inevitable; NFR-06 requires
  machine check.
- **Custom AST script**: rejected — import-linter is the standard
  tool, MIT-licensed, in the allowlist.

### Consequences
- Positive: layering drift caught at commit time.
- Negative: tests that need `sqlalchemy` must live under `repository/`
  or be carefully mocked.

## ADR-016: Configuration via Pydantic settings + 12 `TASKQ_*` env vars

### Status
Accepted

### Satisfies
NFR-04 configuration-handling requirement (SRS §6 NFR-04 specification).

### Context
SPEC §5.1 enumerates the configuration surface (DB URL, CORS origins,
drain timeout, max concurrent, log level, etc.). The configuration
module is an independence module (ADR-002) and must never appear in
logs (T-08 / NFR-04).

### Decision
- `taskq_api.config.get_settings()` (lru_cache) returns a `Settings`
  dataclass populated from `TASKQ_*` env vars.
- 12 documented keys; `.env.example` mirrors the schema.
- Logging filter strips `TASKQ_DB_URL` from any log record.

### Alternatives considered
- **YAML config**: rejected — env-var precedence and container ergonomics
  favor 12-factor env.
- **`dynaconf`**: rejected — adds a dependency for what pydantic-settings
  already does.

### Consequences
- Positive: typed config; one cache; trivial test overrides.
- Negative: env vars are stringly-typed; range validation lives in
  pydantic validators.

## ADR-017: Mutation testing scope (service + repository)

### Status
Accepted

### Satisfies
NFR-08 mutation-testing requirement (SRS §6 NFR-08 specification).

### Context
NFR-08 requires mutation score ≥ 70 on the highest-risk modules.
Running mutmut over the entire codebase would explode runtime; scoping
to the layers where business rules live yields the most signal.

### Decision
- Scope: `taskq_api.service.*` + `taskq_api.repository.*`.
- Threshold: ≥ 70 killed mutants.
- `mutmut` runs against the integration test set; CI fails below
  threshold.

### Alternatives considered
- **Per-file scope**: rejected — fragmentation hides systemic gaps.
- **Whole-codebase**: rejected — `models/` mutations are dominated by
  ORM-generated code; cost vs signal poor.

### Consequences
- Positive: gate is runnable in CI time budget.
- Negative: `api/` mutations are not scored; mitigated by readability
  gate (NFR-11) and integration coverage (NFR-10).

## ADR-018: Verification-of-testable surface (no skipped tests)

### Status
Accepted

### Satisfies
NFR-09 test-assertion requirement (SRS §6 NFR-09 specification); NFR-10
integration-coverage requirement (SRS §6 NFR-10 specification) — integration
tests run end-to-end via `httpx.AsyncClient(transport=ASGITransport(app))`,
covering each error code 401/403/404/409/422/429/503 with one example.

### Context
NFR-09 forbids `pytest.skip` / `skipif` / `xfail` / `--ignore` /
`collect_ignore`. Migration testing must run against a real SQLite file,
not a mock. NFR-10 mandates that integration tests cover the full CRUD
chain, error-code matrix, migration round-trip, rate-limit triggering,
and graceful drain — exercised through the ASGI transport, not by calling
handler functions directly.

### Decision
- Zero skip markers in `03-development/tests/**`.
- `tests/integration/test_migrations_roundtrip.py` performs the
  upgrade/downgrade/upgrade cycle against `sqlite:///./taskq.db`.
- All error codes (401/403/404/409/422/429/500) covered by integration
  tests; missing coverage fails CI.

### Alternatives considered
- **`pytest.mark.skipif(sys.platform == 'win32')`**: rejected — explicit
  NFR-09 prohibition; environment differences are CI's problem, not
  tests'.

### Consequences
- Positive: every CI run exercises the real schema and every handler.
- Negative: CI time grows linearly with FR count; budgeted in
  `.methodology/harness_config.json`.

## ADR-020: Readability / maintainability thresholds (cross-cutting)

### Status
Accepted

### Satisfies
NFR-11 readability requirement (SRS §6 NFR-11 specification) — cross-cutting.

### Context
NFR-11 (per `01-requirements/SRS.md` §6 NFR-11) sets four size and
complexity constraints on the codebase: project MI (LLOC-weighted) ≥ 80,
per-function CC ≤ 10, single file ≤ 400 lines, single directory ≤ 15
files, and each API handler ≤ 40 lines (business logic pushed down to
`service/`). Unlike FR-01..FR-10 or NFR-01..NFR-10, these thresholds are
not owned by a single architectural decision — they are a property of
every ADR above, applied uniformly as code is added.

### Decision
- **MI ≥ 80, CC ≤ 10, file ≤ 400 LOC, dir ≤ 15 files**: enforced
  project-wide by `radon mi -s` and `radon cc -s` in CI; no per-decision
  carve-out exists.
- **Handler ≤ 40 LOC**: enforced by ADR-002 (layering — `api/` is thin)
  and ADR-003 (FastAPI handler pattern); business logic in `service/`.
  Per the SRS specification, the "no handler > 40 lines" rule exists
  because the layering already pushes logic down, not because a single
  decision imposes it.
- **No single owning decision exists for NFR-11**: the readability
  requirement is enforced by the combination of ADR-002 (layering) +
  ADR-003 (thin ASGI handlers) + CI tooling (radon). This ADR records
  that fact so the traceability matrix has a row for NFR-11 instead of
  leaving it uncovered.

### Alternatives considered
- **Distribute NFR-11 across every ADR's "Satisfies" section**: rejected —
  every row would carry an NFR-11 citation that does not reflect a real
  decision; a reader auditing ADR-007 for NFR-11 coverage would find no
  actual decision there. Honest attribution requires a dedicated entry
  noting the cross-cutting nature.
- **Move NFR-11 to TEST_SPEC**: rejected — TEST_SPEC owns the verification
  mechanism (radon invocation), not the architectural decision; the
  decision is "no single decision owns NFR-11" and that decision belongs
  in the ADR file.

### Consequences
- Positive: the traceability matrix now has an honest row for NFR-11
  rather than leaving the requirement uncovered or falsely attributing it
  to one of the decisions above.
- Negative: an auditor reading "ADR-020 satisfies NFR-11" must read this
  ADR to understand that no actual decision implements the requirement —
  the matrix row is a pointer to the explanation, not a single decision
  to review.

## ADR-019: Traceability matrix — ADR ↔ SRS requirement coverage

### Status
Accepted

### Context
Each ADR above cites the FR-IDs and NFR-IDs it satisfies, but the SRS
specification in `01-requirements/SRS.md` is the canonical source for the
FR/NFR list. Without a single traceability matrix that cross-references every
architectural decision to the requirement it satisfies and the SRS section
that establishes it, downstream gate reviewers (Gate 2 architecture review,
Gate 4 quality gate) cannot verify ADR↔SRS coverage mechanically, and a
later audit cannot reconstruct which ADR satisfied which FR without
re-reading every "Context" section. This entry closes that gap by publishing
the traceability matrix in the same document as the decisions themselves so
the specification traceability is co-located with its provenance.

### Decision
The following traceability matrix links each architectural decision to the
requirement it satisfies and the SRS specification it is derived from. Each
row names the decision, the FR/NFR IDs that drive it (the requirement), and
the SRS section that establishes that requirement. The matrix is the single
source of truth for which ADR satisfies which FR/NFR; Gate 2 reviewers must
read this matrix before signing off architecture.

| ADR | Satisfies requirement | Derived from SRS specification |
|-----|-----------------------|---------------------------------|
| ADR-001 (Python 3.11 stdlib-first runtime) | FR-08, NFR-07 | SRS §1 runtime requirement, §6 NFR-07 |
| ADR-002 (Five-layer architecture) | NFR-06 | SRS §6 NFR-06 architecture requirement |
| ADR-003 (ASGI via FastAPI + uvicorn) | FR-01..FR-10, NFR-05 | SRS §3 REST specification, §6 NFR-05 |
| ADR-004 (SQLAlchemy 2.x ORM with `session_scope`) | FR-06, NFR-01, NFR-02, NFR-03 | SRS §4 storage requirement, §6 NFR-01..03 |
| ADR-005 (Alembic-managed schema evolution) | FR-07, NFR-09, NFR-12 | SRS §7 migration specification, §6 NFR-09 |
| ADR-006 (Async subprocess via `create_subprocess_exec`) | FR-02, FR-08, NFR-04 | SRS §3 FR-02/FR-08 specification, §6 NFR-04 |
| ADR-007 (API-key auth SHA-256 + constant-time compare) | FR-03, FR-04, NFR-02 | SRS §3 FR-03/FR-04 requirement, §6 NFR-02 |
| ADR-008 (Token-bucket rate limit + row-level lock) | FR-05 | SRS §3 FR-05 specification |
| ADR-009 (RFC 7807 `ProblemException` error model) | FR-10 | SRS §3 FR-10 specification |
| ADR-010 (Cursor pagination + eager-loaded relations) | FR-01, NFR-01 | SRS §3 FR-01 list requirement, §6 NFR-01 |
| ADR-011 (Redaction layer `service.redaction.scrub`) | NFR-04 | SRS §6 NFR-04 redaction specification |
| ADR-012 (CORS default-deny + credentials disabled) | NFR-02 | SRS §6 NFR-02 requirement |
| ADR-013 (Lifespan-driven runner drain on shutdown) | FR-08 | SRS §3 FR-08 shutdown specification |
| ADR-014 (`make verify-system` exit-gate) | NFR-12 | SRS §8 NFR-12 verification requirement |
| ADR-015 (import-linter layering enforcement) | NFR-06 | SRS §6 NFR-06 layering requirement |
| ADR-016 (Pydantic settings + 12 `TASKQ_*` env vars) | NFR-04 | SRS §5 configuration requirement |
| ADR-017 (Mutation testing scope: service + repository) | NFR-08 | SRS §6 NFR-08 mutation-testing specification |
| ADR-018 (Verification of testable surface, no skips) | NFR-09, NFR-10 | SRS §6 NFR-09 test-assertion requirement, §6 NFR-10 integration-coverage requirement |
| ADR-020 (Readability / maintainability thresholds, cross-cutting) | NFR-11 | SRS §6 NFR-11 readability requirement (cross-cutting; no single owning decision) |

The traceability matrix above is the authoritative link from architectural
decision to SRS requirement. Any future ADR added to this file MUST add a
new row to this matrix in the same commit; ADR updates that change scope
MUST update the corresponding matrix row in the same commit. Failure to do
so is a Gate-2 architecture-review violation (a traceability matrix that
drifts from the decisions it traces is worse than no matrix at all).

### Alternatives considered
- **External traceability table in `01-requirements/TRACEABILITY_MATRIX.md`**:
  rejected — the matrix is most useful when co-located with the decisions it
  traces, and `TRACEABILITY_MATRIX.md` is a meta-document (excluded from
  P2 constitution scans) that already tracks FR↔test mapping, not FR↔ADR.
  Co-locating traceability in this file also keeps the SRS specification
  anchor one section away from the decisions it covers.
- **Inline FR/NFR list per ADR only (no summary table)**: rejected — the
  per-ADR citations are necessary but not sufficient; reviewers still need
  the SRS-specification anchor and the at-a-glance matrix to confirm
  coverage and provenance without re-reading every decision.
- **Machine-readable YAML sidecar (e.g. `adr_traceability.yaml`)**: rejected
  — would create a second source of truth that drifts; the matrix is the
  source.

### Consequences
- Positive: every ADR is now linked to the requirement it satisfies and the
  SRS specification it is derived from; Gate 2 review can spot-check
  coverage against this matrix without re-reading the decisions; a later
  audit can reconstruct which ADR satisfied which FR from one table.
- Negative: the matrix must be updated when an ADR's scope changes; the
  "update in the same commit" rule above is process discipline, not a
  mechanical check, and reviewers must enforce it manually until Gate 2
  adds an automated diff check.
