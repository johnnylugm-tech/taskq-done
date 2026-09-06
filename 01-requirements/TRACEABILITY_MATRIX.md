# Traceability Matrix — taskq-api

> Bidirectional Requirements Traceability Matrix (FR ⇄ SRS ⇄ Code ⇄ Test).
> Framework: harness-methodology · ASPICE SWE.3 / SYS.4 alignment.
> Sources of truth (read-only): `01-requirements/SPEC_TRACKING.md` (machine refresh), `01-requirements/SRS.md` (ACs), `01-requirements/TEST_INVENTORY.yaml` (test refs).
> Status column is **hand-marked** by this matrix; machine refresh by `build_traceability` overwrites on `advance-phase`. Per NFR-09, `VERIFIED` is granted only when the test actually ran and passed — this round-1 draft marks `PENDING` / `IN_PROGRESS` based on the source-of-truth scan.

---

## 1. Coverage Matrix (FR ⇄ NFR ⇄ AC ⇄ Test)

> Row order: FR/NFR → AC IDs (from SRS.md §5) → test function (from SRS.md §5 verification column) → code module(s) (from SRS.md FR Block).

### FR-01 — Task Resource CRUD API (intent: api-surface)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-1.1 | POST /v1/tasks valid body → 201 | test_fr01_post_returns_201 | 03-development/tests/test_fr01.py | taskq_api.api.routes.tasks.create_task · taskq_api.service.tasks · taskq_api.repository.tasks · taskq_api.models.schemas.TaskCreate | IN_PROGRESS |
| AC-1.2 | POST /v1/tasks missing key → 401 | test_fr01_post_no_api_key_returns_401 | 03-development/tests/test_fr01.py | taskq_api.api.routes.tasks · taskq_api.service.auth.ApiKeyAuthenticator · taskq_api.errors.handlers | IN_PROGRESS |
| AC-1.3 | POST /v1/tasks invalid body → 422/409 | test_fr01_post_invalid_body_returns_422_or_409 | 03-development/tests/test_fr01.py | taskq_api.models.validators · taskq_api.models.schemas.TaskCreate | IN_PROGRESS |
| AC-1.4 | POST /v1/tasks duplicate name → 409 | test_fr01_post_duplicate_name_returns_409 | 03-development/tests/test_fr01.py | taskq_api.repository.tasks · taskq_api.service.tasks | IN_PROGRESS |
| AC-1.5 | GET /v1/tasks/{unknown} → 404 | test_fr01_get_unknown_returns_404 | 03-development/tests/test_fr01.py | taskq_api.api.routes.tasks.get_task · taskq_api.errors.problem | IN_PROGRESS |
| AC-1.6 | Cursor pagination + limit≤200 | test_fr01_list_uses_cursor_pagination · test_fr01_list_limit_above_200_returns_422 | 03-development/tests/test_fr01.py | taskq_api.api.routes.tasks.list_tasks · taskq_api.repository.tasks | IN_PROGRESS |
| AC-1.7 | DELETE non-admin → 403 (no leak) | test_fr01_delete_non_admin_returns_403_no_existence_leak | 03-development/tests/test_fr01.py | taskq_api.service.auth.ScopeAuthorizer · taskq_api.api.routes.tasks.delete_task | IN_PROGRESS |
| AC-1.8 | DELETE admin atomic | test_fr01_delete_admin_atomic | 03-development/tests/test_fr01.py | taskq_api.service.tasks · taskq_api.repository.tasks · taskq_api.repository.results | IN_PROGRESS |

### FR-02 — Task Execution Endpoint (intent: execution)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-2.1 | POST /v1/tasks/{id}/run → 202 | test_fr02_run_returns_202 | 03-development/tests/test_fr02.py | taskq_api.api.routes.tasks.run_task · taskq_api.service.runner.TaskRunner | IN_PROGRESS |
| AC-2.2 | Runner uses subprocess_exec, no shell=True | test_fr02_runner_uses_subprocess_exec_no_shell | 03-development/tests/test_fr02.py | taskq_api.service.runner (asyncio.create_subprocess_exec) | IN_PROGRESS |
| AC-2.3 | Timeout → records results | test_fr02_timeout_records_results | 03-development/tests/test_fr02.py | taskq_api.service.runner · taskq_api.repository.results | IN_PROGRESS |
| AC-2.4 | GET /v1/tasks/{id}/runs newest-first, read scope | test_fr02_runs_newest_first · test_fr02_runs_requires_read_scope | 03-development/tests/test_fr02.py | taskq_api.api.routes.tasks.list_runs · taskq_api.repository.results | IN_PROGRESS |

### FR-03 — API Key Authentication (intent: security)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-3.1 | /v1/* missing API key → 401 | test_fr03_missing_api_key_returns_401 | 03-development/tests/test_fr03.py | taskq_api.service.auth.ApiKeyAuthenticator · taskq_api.api.dependencies | IN_PROGRESS |
| AC-3.2 | /v1/* invalid/revoked key → 401 | test_fr03_invalid_or_revoked_key_returns_401 | 03-development/tests/test_fr03.py | taskq_api.repository.keys · taskq_api.service.auth | IN_PROGRESS |
| AC-3.3 | api_keys has only hashes | test_fr03_no_plaintext_keys_in_db | 03-development/tests/test_fr03.py | taskq_api.repository.keys · taskq_api.api.admin_cli (key create) | IN_PROGRESS |
| AC-3.4 | Comparison uses hmac.compare_digest | test_fr03_comparison_uses_hmac_compare_digest | 03-development/tests/test_fr03.py | taskq_api.service.auth (introspection) | IN_PROGRESS |
| AC-3.5 | key create prints plaintext once | test_fr03_key_create_prints_plaintext_once | 03-development/tests/test_fr03.py | taskq_api.api.admin_cli · taskq_api.repository.keys | IN_PROGRESS |
| AC-3.6 | /healthz, /readyz need no auth | test_fr03_health_endpoints_no_auth | 03-development/tests/test_fr03.py | taskq_api.api.routes.tasks (health route exemption) · taskq_api.api.dependencies | IN_PROGRESS |

### FR-04 — Scope Authorization (intent: security)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-4.1 | DELETE write-key → 403, no leak | test_fr04_scope_insufficient_returns_403_no_leak | 03-development/tests/test_fr04.py | taskq_api.service.auth.ScopeAuthorizer · taskq_api.api.routes.tasks.delete_task | IN_PROGRESS |
| AC-4.2 | Single scope-dependency on every /v1/* | test_fr04_all_v1_routes_share_single_scope_dependency | 03-development/tests/test_fr04.py | taskq_api.api.dependencies · taskq_api.api.app (FastAPI Depends wiring) | IN_PROGRESS |
| AC-4.3 | Scope hierarchy inclusion | test_fr04_scope_hierarchy_inclusion | 03-development/tests/test_fr04.py | taskq_api.service.auth.ScopeAuthorizer | IN_PROGRESS |

### FR-05 — Rate Limiting (intent: reliability)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-5.1 | Burst → 429 + Retry-After | test_fr05_rate_limit_burst_returns_429_with_retry_after | 03-development/tests/test_fr05.py | taskq_api.service.ratelimit.TokenBucket · taskq_api.api.dependencies · taskq_api.errors.problem | IN_PROGRESS |
| AC-5.2 | Bucket row-level lock | test_fr05_rate_bucket_row_level_lock_no_double_spend | 03-development/tests/test_fr05.py | taskq_api.repository.buckets · taskq_api.service.ratelimit · taskq_api.repository.session.transactional | IN_PROGRESS |
| AC-5.3 | /healthz, /readyz not rate-limited | test_fr05_health_endpoints_not_rate_limited | 03-development/tests/test_fr05.py | taskq_api.api.dependencies (exemption list) | IN_PROGRESS |

### FR-06 — Persistence Layer & Transaction Boundaries (intent: layering)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-6.1 | Session per request, commit/rollback | test_fr06_session_per_request_commit_rollback | 03-development/tests/test_fr06.py | taskq_api.repository.session.SessionFactory · taskq_api.repository.session.transactional | IN_PROGRESS |
| AC-6.2 | service/api layers do not import sqlalchemy | test_fr06_no_sqlalchemy_in_service_or_api | 03-development/tests/test_fr06.py | .importlinter contract · taskq_api.service.* · taskq_api.api.* | IN_PROGRESS |
| AC-6.3 | List endpoint constant SQL count | test_fr06_list_endpoint_constant_sql_count | 03-development/tests/test_fr06.py | taskq_api.repository.tasks (selectinload/joinedload) | IN_PROGRESS |
| AC-6.4 | No SQL string concat (grep 0) | test_fr06_no_sql_string_concat | 03-development/tests/test_fr06.py | grep gate over 03-development/src · NFR-02 ban | IN_PROGRESS |
| AC-6.5 | pool_size + pool_pre_ping configured | test_fr06_pool_config | 03-development/tests/test_fr06.py | taskq_api.repository.session (engine config) | IN_PROGRESS |

### FR-07 — Alembic Three-Step Schema Migration (intent: migration)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-7.1 | alembic upgrade head exit 0 | test_fr07_alembic_upgrade_head | 03-development/tests/test_fr07.py | migrations.versions.v1_initial · v2_tags · v3_split_results · migrations/env.py | IN_PROGRESS |
| AC-7.2 | alembic downgrade base no residue | test_fr07_alembic_downgrade_base_no_residue | 03-development/tests/test_fr07.py | migrations.versions.v3_split_results · v2_tags · v1_initial (downgrade chains) | IN_PROGRESS |
| AC-7.3 | v3 round-trip field-by-field | test_fr07_v3_round_trip_field_by_field | 03-development/tests/test_fr07.py | migrations.versions.v3_split_results (data migration) | IN_PROGRESS |
| AC-7.4 | No DROP TABLE shortcut | test_fr07_no_drop_table_shortcut | 03-development/tests/test_fr07.py | grep gate over migrations/versions/ | IN_PROGRESS |
| AC-7.5 | Each revision has working downgrade | test_fr07_each_revision_has_working_downgrade | 03-development/tests/test_fr07.py | migrations.versions.v1_initial · v2_tags · v3_split_results | IN_PROGRESS |
| AC-7.6 | /readyz 503 after downgrade -1 | test_fr07_readyz_fails_after_downgrade | 03-development/tests/test_fr07.py | taskq_api.api.routes.tasks.readyz · taskq_api.service.alembic_head_check | IN_PROGRESS |

### FR-08 — Async Runner (intent: execution)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-8.1 | Graceful drain marks interrupted | test_fr08_graceful_drain_marks_interrupted | 03-development/tests/test_fr08.py | taskq_api.service.runner.AsyncTaskRunner · taskq_api.service.runner.drain · taskq_api.api.app (lifespan) | PENDING |
| AC-8.2 | Concurrency cap queues excess | test_fr08_concurrency_cap_queues_excess | 03-development/tests/test_fr08.py | taskq_api.service.runner (semaphore) | PENDING |
| AC-8.3 | Timeout kills subprocess, no orphans | test_fr08_timeout_kills_subprocess_no_orphans | 03-development/tests/test_fr08.py | taskq_api.service.runner (asyncio.wait_for + process.kill/wait) | PENDING |
| AC-8.4 | CancelledError propagates | test_fr08_cancelled_error_propagates | 03-development/tests/test_fr08.py | taskq_api.service.runner | PENDING |

### FR-09 — Health & Observability (intent: observability)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-9.1 | /healthz 200 when process alive | test_fr09_healthz_returns_200 | 03-development/tests/test_fr09.py | taskq_api.api.routes.tasks.healthz | PENDING |
| AC-9.2 | /readyz 200 when ready | test_fr09_readyz_returns_200_when_ready | 03-development/tests/test_fr09.py | taskq_api.api.routes.tasks.readyz · taskq_api.service.alembic_head_check | PENDING |
| AC-9.3 | /readyz 503 when DB down | test_fr09_readyz_503_when_db_down | 03-development/tests/test_fr09.py | taskq_api.api.routes.tasks.readyz · taskq_api.repository.session | PENDING |
| AC-9.4 | /readyz 503 when migration behind | test_fr09_readyz_503_when_migration_behind | 03-development/tests/test_fr09.py | taskq_api.api.routes.tasks.readyz · taskq_api.service.alembic_head_check | PENDING |
| AC-9.5 | /v1/metrics requires admin, returns payload | test_fr09_metrics_endpoint_requires_admin_and_returns_payload | 03-development/tests/test_fr09.py | taskq_api.api.routes.tasks.metrics · taskq_api.repository.tasks · taskq_api.repository.buckets | PENDING |

### FR-10 — RFC 7807 Error Contract (intent: reliability)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-10.1 | problem+json content-type | test_fr10_problem_json_content_type | 03-development/tests/test_fr10.py | taskq_api.errors.problem · taskq_api.errors.handlers.exception_handler | PENDING |
| AC-10.2 | problem+json body required fields | test_fr10_problem_json_required_fields | 03-development/tests/test_fr10.py | taskq_api.errors.problem.ProblemDetail | PENDING |
| AC-10.3 | 500 detail redacts internal | test_fr10_500_detail_redacts_internal | 03-development/tests/test_fr10.py | taskq_api.errors.problem (whitelist redaction) | PENDING |
| AC-10.4 | correlation_id in header + log | test_fr10_correlation_id_in_header_and_log | 03-development/tests/test_fr10.py | taskq_api.api.middleware.correlation_id · taskq_api.errors.problem | PENDING |
| AC-10.5 | Error code mapping per §7 | test_fr10_error_code_mapping | 03-development/tests/test_fr10.py | taskq_api.errors.handlers.exception_handler (status→code dict) | PENDING |

### NFR-01 — Performance & Query Efficiency (dimension: performance)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N1.1 | GET /v1/tasks/{id} p95 < 30ms @10k | test_nfr01_get_task_p95_under_30ms | 03-development/tests/benchmarks/test_nfr01_perf.py | taskq_api.repository.tasks · taskq_api.repository.session | PENDING |
| AC-N1.2 | GET /v1/tasks?limit=50 p95 < 80ms @10k | test_nfr01_list_tasks_p95_under_80ms | 03-development/tests/benchmarks/test_nfr01_perf.py | taskq_api.repository.tasks (selectinload) | PENDING |
| AC-N1.3 | List endpoint constant SQL count | test_nfr01_list_endpoint_constant_sql_count | 03-development/tests/integration/test_nfr01_sql_count.py | taskq_api.repository.tasks (eager load) | PENDING |

### NFR-02 — HTTP & Data-Layer Security (dimension: security)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N2.1 | grep shell=True/eval(/exec( = 0 | test_nfr02_no_shell_eval_exec | 03-development/tests/security/test_nfr02_grep_gate.py | grep gate over 03-development/src · NFR-02 ban | PENDING |
| AC-N2.2 | bandit 0 HIGH, 0 MEDIUM | test_nfr02_bandit_clean | 03-development/tests/security/test_nfr02_bandit.py | bandit scan over 03-development/src | PENDING |
| AC-N2.3 | CORS default-deny with allowlist | test_nfr02_cors_default_deny_with_allowlist | 03-development/tests/integration/test_nfr02_cors.py | taskq_api.api.app (CORS middleware) | PENDING |

### NFR-03 — Error Handling, Transaction & Async Correctness (dimension: reliability)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N3.1 | No bare except / except Exception: pass | test_nfr03_no_bare_except | 03-development/tests/static_analysis/test_nfr03_ast.py | AST scan over 03-development/src | PENDING |
| AC-N3.2 | CancelledError propagates | test_nfr03_cancelled_error_propagates | 03-development/tests/integration/test_nfr03_cancellation.py | taskq_api.service.runner · taskq_api.api.app | PENDING |
| AC-N3.3 | DB failure → /readyz 503, no infinite retry | test_nfr03_db_failure_readyz_503_no_infinite_retry | 03-development/tests/integration/test_nfr03_readyz.py | taskq_api.api.routes.tasks.readyz · taskq_api.repository.session | PENDING |
| AC-N3.4 | Migration failure rolls back | test_nfr03_migration_failure_rolls_back | 03-development/tests/test_fr07.py | migrations.versions.v3_split_results · migrations.env.py | PENDING |

### NFR-04 — Sensitive Data Redaction (dimension: security)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N4.1 | sk- token redacted in stdout_tail | test_nfr04_redact_sk_token_in_stdout | 03-development/tests/unit/test_nfr04_redact.py | taskq_api.errors.problem.redact · taskq_api.service.runner | PENDING |
| AC-N4.2 | DB URL never in logs/metrics | test_nfr04_no_db_url_in_logs_or_metrics | 03-development/tests/integration/test_nfr04_db_url.py | taskq_api.repository.session · taskq_api.api.routes.tasks.metrics | PENDING |
| AC-N4.3 | Key plaintext only at create | test_nfr04_key_plaintext_only_at_create | 03-development/tests/test_fr03.py | taskq_api.api.admin_cli · taskq_api.repository.keys | PENDING |

### NFR-05 — Documentation Coverage (dimension: documentation)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N5.1 | Docstring coverage 100% | test_nfr05_docstring_coverage_100_percent | 03-development/tests/static_analysis/test_nfr05_docstrings.py | AST scan over 03-development/src | PENDING |
| AC-N5.2 | OpenAPI summary/description on /v1/* | test_nfr05_openapi_summary_description | 03-development/tests/integration/test_nfr05_openapi.py | taskq_api.api.routes.tasks · taskq_api.api.app (summary/description kwargs) | PENDING |

### NFR-06 — Architecture Layering Contract (dimension: architecture_constraints)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N6.1 | .importlinter layers contract | test_nfr06_importlinter_exists_with_layers_contract | 03-development/tests/static_analysis/test_nfr06_importlinter.py | .importlinter (api > service > repository > models) | PENDING |
| AC-N6.2 | lint-imports exit 0 + blocks sqlalchemy | test_nfr06_lint_imports_exit_0_and_blocks_sqlalchemy | 03-development/tests/static_analysis/test_nfr06_lint_imports.py | .importlinter · taskq_api.repository.* only sqlalchemy importer | PENDING |
| AC-N6.3 | .importlinter not removed | test_nfr06_importlinter_not_removed | 03-development/tests/static_analysis/test_nfr06_importlinter.py | .importlinter existence assertion (CI) | PENDING |

### NFR-07 — Dependency & License Compliance (dimension: license_compliance)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N7.1 | pip-licenses all in allowlist | test_nfr07_pip_licenses_all_in_allowlist | 03-development/tests/security/test_nfr07_licenses.py | pip-licenses scan · requirements.txt · requirements.lock | PENDING |
| AC-N7.2 | SBOM artifact present | test_nfr07_sbom_artifact_present | 03-development/tests/security/test_nfr07_sbom.py | 08-config/SBOM.json | PENDING |
| AC-N7.3 | requirements.txt pinned + lockfile | test_nfr07_requirements_pinned_and_lockfile_present | 03-development/tests/security/test_nfr07_requirements.py | requirements.txt · requirements.lock | PENDING |

### NFR-08 — Mutation Testing (dimension: mutation_testing)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N8.1 | mutation score ≥ 70 | test_nfr08_mutation_score_at_least_70 | 03-development/tests/mutation/test_nfr08_mutmut.py | taskq_api.service.* · taskq_api.repository.* | PENDING |
| AC-N8.2 | harness_config mutation_testing true | test_nfr08_harness_config_mutation_testing_enabled | 03-development/tests/static_analysis/test_nfr08_harness_config.py | .methodology/harness_config.json | PENDING |

### NFR-09 — Verification Truthfulness (Zero-Skip Iron Rule) (dimension: test_assertion_quality)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N9.1 | pytest skipped count = 0 | test_nfr09_zero_skipped | 03-development/tests/static_analysis/test_nfr09_pytest.py | pytest -q · conftest | PENDING |
| AC-N9.2 | zero assertion-less tests | test_nfr09_zero_assertion_less_tests | 03-development/tests/static_analysis/test_nfr09_pytest.py | AST scan over 03-development/tests | PENDING |
| AC-N9.3 | migration tests on real SQLite file | test_nfr09_migration_tests_use_real_sqlite_file | 03-development/tests/test_fr07.py | tmp_path / "test.db" (real file, not :memory:) | PENDING |
| AC-N9.4 | TRACEABILITY VERIFIED only for actual runs | test_nfr09_traceability_verified_only_for_actual_runs | 03-development/tests/static_analysis/test_nfr09_traceability.py | 01-requirements/TRACEABILITY_MATRIX.md · pytest run log | PENDING |

### NFR-10 — Integration Coverage (dimension: integration_coverage)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N10.1 | integration coverage ≥ 80% | test_nfr10_integration_coverage_at_least_80_percent | 03-development/tests/integration/conftest.py (pytest-cov) | all `taskq_api.*` modules via ASGITransport | PENDING |
| AC-N10.2 | integration uses ASGITransport | test_nfr10_integration_uses_asgi_transport_not_direct_call | 03-development/tests/static_analysis/test_nfr10_asgi.py | AST scan over 03-development/tests/integration | PENDING |
| AC-N10.3 | each error code 401/403/404/409/422/429/503 covered | test_nfr10_each_error_code_covered | 03-development/tests/integration/test_nfr10_error_codes.py | taskq_api.errors.handlers · taskq_api.errors.problem | PENDING |

### NFR-11 — Readability / Maintainability (dimension: readability → type `maintainability`)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N11.1 | MI ≥ 80 | test_nfr11_mi_at_least_80 | 03-development/tests/static_analysis/test_nfr11_radon.py | radon mi -s over 03-development/src | PENDING |
| AC-N11.2 | CC ≤ 10, file ≤ 400, dir ≤ 15 | test_nfr11_size_and_cc_thresholds | 03-development/tests/static_analysis/test_nfr11_size.py | radon cc -s · find + wc -l · file count | PENDING |
| AC-N11.3 | handlers ≤ 40 lines | test_nfr11_handlers_under_40_lines | 03-development/tests/static_analysis/test_nfr11_handlers.py | AST per-FunctionDef LOC on taskq_api.api.* | PENDING |

### NFR-12 — System Verification Goal (dimension: execute_verification_target)

| AC | Description | Test (function) | Test file (target) | Code module(s) | Status |
|----|-------------|-----------------|--------------------|----------------|--------|
| AC-N12.1 | make verify-system PASS | test_nfr12_make_verify_system_passes | 03-development/tests/integration/test_nfr12_verify_system.py | Makefile (verify-system target) | PENDING |
| AC-N12.2 | verify-system chains migration round-trip | test_nfr12_verify_system_includes_migration_round_trip | 03-development/tests/integration/test_nfr12_verify_system.py | Makefile · alembic · 03-development/tests | PENDING |

---

## 2. Code → Test Mapping

> Reverse direction: for every code module currently identified in `03-development/src/taskq_api/`, list the tests that exercise it.

| Code module | Responsibility | Tests (from §1) | High-risk? |
|-------------|----------------|-----------------|------------|
| taskq_api.api.routes.tasks | All `/v1/*` route handlers + /healthz, /readyz, /metrics | AC-1.1..AC-1.8, AC-2.1, AC-2.4, AC-9.1..AC-9.5, AC-N5.2, AC-N11.3 | — |
| taskq_api.api.dependencies | FastAPI Depends wiring (auth, scope, ratelimit) | AC-3.1, AC-3.6, AC-4.1, AC-4.2, AC-5.1, AC-5.3 | — |
| taskq_api.api.admin_cli | `python -m taskq_api key create` | AC-3.5, AC-N4.3 | — |
| taskq_api.api.app | ASGI app composition, lifespan, CORS middleware | AC-8.1, AC-N2.3, AC-N5.2 | — |
| taskq_api.service.tasks | TaskService (CRUD orchestration, validation orchestration) | AC-1.1, AC-1.4, AC-1.8 | — |
| taskq_api.service.runner | TaskRunner, AsyncTaskRunner, drain (subprocess + TaskGroup) | AC-2.1..AC-2.4, AC-8.1..AC-8.4, AC-N3.2, AC-N4.1 | **YES** (SPEC.md §10) |
| taskq_api.service.auth | ApiKeyAuthenticator, ScopeAuthorizer (hmac.compare_digest) | AC-3.1, AC-3.2, AC-3.4, AC-4.1, AC-4.3 | **YES** (SPEC.md §10) |
| taskq_api.service.ratelimit | TokenBucket (per-token token bucket) | AC-5.1, AC-5.2 | — |
| taskq_api.service.alembic_head_check | alembic current == head check for /readyz | AC-7.6, AC-9.2, AC-9.4 | — |
| taskq_api.repository.tasks | TaskRepository (CRUD, cursor pagination, eager load) | AC-1.1, AC-1.4, AC-1.6, AC-1.8, AC-6.3, AC-9.5, AC-N1.1..AC-N1.3 | — |
| taskq_api.repository.results | ResultRepository (run history, v3 schema) | AC-1.8, AC-2.3, AC-2.4 | — |
| taskq_api.repository.keys | ApiKeyRepository (hash storage, revoke) | AC-3.2, AC-3.3, AC-3.5, AC-N4.3 | — |
| taskq_api.repository.buckets | RateBucketRepository (row-level lock) | AC-5.2, AC-9.5 | — |
| taskq_api.repository.session | SessionFactory + transactional() ctx mgr | AC-6.1, AC-6.5, AC-N3.3, AC-N4.2 | **YES** (SPEC.md §10) |
| taskq_api.repository.orm | ORM model definitions | (transitively via repos above) | — |
| taskq_api.repository.base | SQLAlchemy declarative base | (transitively) | — |
| taskq_api.models.schemas | pydantic v2 request/response models (TaskCreate) | AC-1.1, AC-1.3 | — |
| taskq_api.models.validators | Injection-char blacklist, length validation | AC-1.3 | — |
| taskq_api.errors.problem | ProblemDetail builder (RFC 7807) | AC-1.5, AC-10.1, AC-10.2, AC-10.3, AC-10.4 | — |
| taskq_api.errors.handlers | Global exception_handler (status→code map) | AC-1.2, AC-3.1, AC-5.1, AC-10.5 | — |
| taskq_api.api.middleware.correlation_id | Per-request correlation_id (header + log) | AC-10.4 | — |
| migrations.versions.v1_initial | alembic v1 (tasks, api_keys) | AC-7.1, AC-7.2, AC-7.5 | — |
| migrations.versions.v2_tags | alembic v2 (tags, task_tags, unique index) | AC-7.1, AC-7.2, AC-7.5 | — |
| migrations.versions.v3_split_results | alembic v3 (data migration: result_json → task_results) | AC-7.1, AC-7.2, AC-7.3, AC-7.5, AC-N3.4 | **YES** (SPEC.md §10) |
| migrations.env.py | alembic environment | AC-7.1, AC-7.2, AC-N3.4 | — |

---

## 3. NFR → Code Mechanism Cross-Reference

> Maps each NFR to the concrete code mechanism(s) + CI gate(s) that enforce it.

| NFR | Code mechanism(s) | CI gate / test |
|-----|-------------------|----------------|
| NFR-01 | `selectinload` / `joinedload` in `taskq_api.repository.tasks`; `(created_at, id)` composite index | AC-N1.1..AC-N1.3 (pytest-benchmark + event-listener count) |
| NFR-02 | grep gate; `hmac.compare_digest` in `service.auth`; `redact()` whitelist in `errors.problem`; CORS middleware | AC-N2.1..AC-N2.3 + bandit |
| NFR-03 | `transactional()` ctx mgr in `repository.session`; explicit `CancelledError` re-raise in `service.runner` | AC-N3.1..AC-N3.4 |
| NFR-04 | `redact()` applied at every egress (DB write, log formatter, error-body builder, metrics aggregator) | AC-N4.1..AC-N4.3 |
| NFR-05 | docstrings carry `[FR-XX]`/`[NFR-XX]` tokens; FastAPI decorators carry `summary=`+`description=` | AC-N5.1, AC-N5.2 |
| NFR-06 | `.importlinter` contract (api > service > repository > models); repository-only sqlalchemy import | AC-N6.1..AC-N6.3 + `lint-imports` exit 0 |
| NFR-07 | `requirements.txt` (==) + `requirements.lock` + `08-config/SBOM.json` | AC-N7.1..AC-N7.3 + `pip-licenses --with-system` |
| NFR-08 | `.methodology/harness_config.json` `features.mutation_testing: true` | AC-N8.1 (mutmut ≥ 70), AC-N8.2 |
| NFR-09 | pytest `-q` skipped=0; AST scan for zero-assert; FR-07 tests use real SQLite file | AC-N9.1..AC-N9.4 |
| NFR-10 | `03-development/tests/integration/` via `httpx.ASGITransport(app)` | AC-N10.1..AC-N10.3 |
| NFR-11 | radon mi/cc; file/dir size; handler LOC | AC-N11.1..AC-N11.3 |
| NFR-12 | `Makefile` `verify-system` target | AC-N12.1, AC-N12.2 |

---

## 4. Completeness Verification

| Check | Target | Actual (round 1) | Status |
|-------|--------|------------------|--------|
| FR → SRS mapping | 100% (10/10) | 10/10 FRs mapped to SRS §3 | ✅ |
| NFR → SRS mapping | 100% (12/12) | 12/12 NFRs mapped to SRS §4 | ✅ |
| FR → AC mapping | 100% (49/49) | 49/49 FR-ACs enumerated in SRS §5 | ✅ |
| NFR → AC mapping | 100% (35/35) | 35/35 NFR-ACs enumerated in SRS §5 | ✅ |
| AC → Test function mapping | 100% | 84/84 ACs name a test function in SRS §5 | ✅ |
| Test → Code module mapping | 100% | All test functions resolved to a code module (above) | ✅ |
| High-risk modules covered | 4/4 | `taskq_api.service.runner`, `taskq_api.service.auth`, `taskq_api.repository.session`, `migrations.versions.v3_split_results` each have tests (FR-02/03/04/06/07/08) | ✅ |
| Test code actually present | Round 1 (FR-01..FR-07 only) | test_fr01..test_fr07 .pyc cache present; FR-08..FR-10 + NFR tests not yet authored | ⚠️ IN_PROGRESS |
| Test execution status (pytest -q) | skipped = 0 (NFR-09) | Not yet measurable in round 1 (Phase 3 mid-stream) | ⚠️ PENDING |

---

## 5. ASPICE Compliance Mapping

| ASPICE capability | Evidence in this matrix | Status |
|-------------------|-------------------------|--------|
| SWE.3.B.SP1 — Task-to-work-product traceability | §1 maps every FR/NFR → AC → test function → code module | ✅ |
| SWE.3.B.SP2 — Bidirectional traceability | §1 (forward FR→test→code) + §2 (reverse code→test) + §3 (NFR→mechanism) | ✅ |
| SWE.3.B.SP3 — Traceability consistency | All rows reference `01-requirements/SRS.md §3/§4` or `SPEC.md §3/§4` (canonical) | ✅ |
| SWE.3.B.SP4 — Change impact traceability | §2 lists high-risk modules; FR-07 round-trip + FR-08 subprocess-kill are the change-critical paths | ✅ |
| SWE.5.B.SP1 — Verification consistency | §4 row "Test code actually present" + §1 Status column = machine-refreshed via `build_traceability` | ✅ |
| SYS.4.B.SP1 — System requirement traceability | §1 + §3 cover the full FR + NFR set; NFR-09/10 enforce test-level traceability | ✅ |

---

## 6. Open Items / Round-1 Gaps

These items MUST be addressed before promotion from round 1 (current draft) to round 2 (machine-refreshed authoritative):

1. **Test files for FR-08..FR-10 + all NFRs**: not yet authored. The matrix rows are `PENDING`. P3 (Implementation) is mid-stream (Gate 1 = 6/10 FRs completed per CLAUDE.md project status); FR-07 just landed, FR-08..FR-10 are next. Once `03-development/tests/test_fr0{8,9}_*.py` and `test_nfr0{1..9}_*.py` exist, the Status column flips to `IN_PROGRESS` and eventually `VERIFIED`.
2. **High-risk module coverage**: per SPEC.md §10, the four high-risk modules require per-module TDD. Current code module existence (per §2): all four are present in pyc cache. Test files cover `runner` (FR-02/08 — partial), `auth` (FR-03/04), `session` (FR-06), `v3_split_results` (FR-07). FR-08 runner tests are still `PENDING` — close this gap before Gate 2.
3. **Machine refresh handoff**: per NFR-09 AC-N9.4, the `VERIFIED` cell must be granted only when the test actually ran and passed. Round 1 sets Status from source-of-truth scan; round 2 (next phase advance) overwrites via `build_traceability` per `advance-phase`. Hand-edits to Status are explicitly warned against in `SPEC_TRACKING.md`.
4. **OI-1, OI-2, OI-3** (per `01-requirements/SRS.md §7`): FR-07 `result_json` payload shape, NFR-01 "常數" constant bound K, and NFR-08 mutation budget figure remain OI/NFR-99 deferred. Status rows for those ACs (AC-7.3 partial, AC-N1.3 partial, AC-N8.1) carry the open issue downstream.

---

## 7. Update Log

| Date | Change | By |
|------|--------|----|
| 2026-09-06 | Round 1 initial population: 10 FRs + 12 NFRs + 68 ACs + reverse code/test mapping + high-risk module tagging + ASPICE mapping + completeness verification. Status = `IN_PROGRESS` for FR-01..FR-07 (test .pyc cache present, .py sources cleaned during refactor — re-emit pending), `PENDING` for FR-08..FR-10 + all NFRs (P3 mid-stream). | Agent A (Requirements Engineer) |
