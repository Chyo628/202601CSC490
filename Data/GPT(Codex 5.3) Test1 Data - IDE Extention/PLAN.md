# Mini Celery (Python 3.12) — Decision-Complete Build Plan

## Summary
- Build a greenfield repository named `mini_celery` with four runnables: FastAPI API, Redis Streams worker, scheduler, and CLI.
- Use Redis Streams + consumer groups for at-least-once delivery and reclaim via `XPENDING` + `XCLAIM`.
- Persist authoritative job state and transition history in SQLite via SQLAlchemy 2.0.
- Provide deterministic behavior via explicit UTC timestamps, injectable clock, bounded jitter, strict state machine, and structured error paths.
- Ship Docker Compose startup (`redis`, `api`, `worker`, `scheduler`) plus `make` targets and script fallbacks.
- Include unit + integration tests that directly cover all required reliability scenarios.

## Repository Layout
- `pyproject.toml` with dependencies, ruff, pytest, mypy config.
- `Dockerfile` single image used by API/worker/scheduler/cli.
- `docker-compose.yml` for local runtime.
- `docker-compose.test.yml` for isolated integration test execution.
- `Makefile` with `test`, `lint`, `up`, `down`, `loadtest`.
- `scripts/dev.py` fallback task runner for environments without `make`.
- `README.md` with architecture, quickstart, failure scenarios, tradeoffs, assumptions, extensibility.
- `config/schedules.yaml` recurring schedule examples.
- `mini_celery/__init__.py`
- `mini_celery/config.py` env settings.
- `mini_celery/logging.py` JSON logging setup.
- `mini_celery/metrics.py` Prometheus counters/gauges/histograms.
- `mini_celery/db.py` engine/session/init.
- `mini_celery/models.py` SQLAlchemy models and enums.
- `mini_celery/schemas.py` Pydantic request/response models.
- `mini_celery/state_machine.py` allowed transitions + validation.
- `mini_celery/repository.py` DB operations (jobs, events, idempotency, schedule ticks).
- `mini_celery/broker/base.py` broker protocol.
- `mini_celery/broker/redis_streams.py` stream/group operations, reclaim, dlq.
- `mini_celery/tasks/base.py` task protocol + registry.
- `mini_celery/tasks/types.py` `sleep`, `http_get`, `cpu_burn`, `fail_then_succeed`, `pipeline_demo`.
- `mini_celery/worker/main.py` worker service.
- `mini_celery/scheduler/main.py` delayed + recurring scheduler service.
- `mini_celery/api/main.py` FastAPI app wiring.
- `mini_celery/api/routes_jobs.py` jobs/admin/events endpoints.
- `mini_celery/api/routes_health.py` `/healthz`.
- `mini_celery/cli.py` submit/status/tail/loadtest.
- `tests/unit/...`
- `tests/integration/...`
- `tests/conftest.py`

## Public APIs, Interfaces, and Types
- HTTP API (required + one additive endpoint):
1. `POST /jobs` exact request contract; returns `{ "job_id": "<uuid>" }`.
2. `GET /jobs/{job_id}` returns full job record.
3. `POST /jobs/{job_id}/cancel` supports queued/delayed cancellation and running cancel request.
4. `POST /jobs/{job_id}/retry` allowed only for `FAILED`/`DEAD`; resets `attempts=0` and requeues.
5. `GET /jobs?state=&type=&limit=&cursor=` cursor pagination.
6. `GET /metrics` Prometheus exposition.
7. `GET /healthz` liveness + Redis + DB checks.
8. `GET /jobs/{job_id}/events?cursor_ts=` additive endpoint for CLI tail polling.
- Redis stream message schema:
- Fields: `job_id`, `type`, `queue`, `enqueued_at`, `dispatch_token`.
- Streams: `jobs:{queue}` and DLQ stream `jobs:dlq`.
- Group: `workers`.
- Scheduler config schema (`config/schedules.yaml`):
- `id`, `cron`, `queue`, `type`, `payload`, `max_attempts`, `enabled`.
- CLI commands:
- `submit`, `status`, `list`, `tail-events`, `loadtest`.

## Data Model
- Required `jobs` table with all requested columns.
- Required `job_events` append-only table with requested columns.
- Added columns in `jobs` for reliability:
- `queue` (string), `cancel_requested` (bool), `lease_expires_at` (datetime nullable), `last_stream_id` (string nullable), `enqueued` (bool), `version` (int optimistic concurrency).
- Added table `idempotency_locks`:
- `key` (PK), `owner_job_id`, `status` (`IN_PROGRESS|SUCCEEDED`), `result` JSON, `updated_at`.
- Added table `schedule_ticks`:
- `schedule_id`, `planned_time`, `job_id`, `created_at`; unique `(schedule_id, planned_time)`.

## State Machine and Core Semantics
- States: `QUEUED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `CANCELED`, `DEAD`.
- Allowed transitions:
- `QUEUED -> RUNNING|CANCELED`
- `RUNNING -> SUCCEEDED|FAILED|CANCELED|DEAD|QUEUED` (queued only for retryable failures/backoff)
- `FAILED -> QUEUED` (manual retry)
- `DEAD -> QUEUED` (manual retry)
- Terminal states: `SUCCEEDED`, `FAILED`, `CANCELED`, `DEAD`.
- Every transition appends one immutable `job_events` row in same DB transaction as state update.

## Worker Behavior
- Concurrency model: asyncio tasks (`WORKER_CONCURRENCY`, default `4`) in one process.
- Consume loop: `XREADGROUP BLOCK` on `jobs:{queue}`.
- Reclaim loop: every `RECLAIM_INTERVAL_SEC` run `XPENDING` filter by idle > `VISIBILITY_TIMEOUT_SEC`, then `XCLAIM`.
- Processing order:
1. Receive stream message.
2. Load job and atomically claim execution if runnable.
3. Persist `RUNNING` and increment attempt count.
4. Execute task with payload validation.
5. Persist final/retry state + event.
6. `XACK` only after successful durable persistence of final transition.
- Retry policy:
- Backoff `delay = min(cap, base * multiplier^(attempt-1)) + jitter`.
- Jitter bounded by `jitter_ratio` (default `0.2`) and deterministic in tests via seeded RNG fixture.
- Retryable failures re-enter `QUEUED` with `next_run_at = now + delay` and `enqueued=false`.
- Exceeded attempts -> `DEAD` and publish entry to `jobs:dlq`.
- Idempotency:
- API dedupe for same key while non-terminal.
- Worker checks `idempotency_locks` before side effects.
- If key already `SUCCEEDED`, worker short-circuits and reuses stored result.
- First executor acquires/owns lock; success stores result in lock row.
- Cooperative cancellation:
- `pipeline_demo` executes step-wise and checks `cancel_requested` between steps.
- On cancellation request during running, transition to `CANCELED` at next checkpoint.

## Scheduler Behavior
- Delayed jobs:
- Poll DB for `state=QUEUED`, `enqueued=false`, `next_run_at<=now`, `cancel_requested=false`.
- Enqueue to Redis stream and set `enqueued=true`, `last_stream_id`.
- If crash after enqueue before DB update, duplicate enqueue is tolerated; worker CAS logic prevents duplicate execution.
- Recurring jobs:
- Load schedules from YAML/JSON on startup.
- Use croniter in UTC to compute planned ticks.
- For each planned tick, insert `schedule_ticks` unique key `(schedule_id, planned_time)`.
- Only successful insert creates job and increments `scheduler_enqueues_total{schedule_id}`.
- Restart-safe dedupe comes from `schedule_ticks` uniqueness.

## Observability
- Structured JSON logs for API/worker/scheduler/CLI.
- Mandatory correlation fields emitted when available: `job_id`, `stream_id`, `consumer`, `attempt`, `state`.
- Prometheus metrics exposed at `/metrics` with required metrics:
- `jobs_enqueued_total{type,queue}`
- `jobs_started_total{type}`
- `jobs_finished_total{type,state}`
- `job_duration_seconds{type}` histogram
- `queue_depth{queue}` gauge
- `worker_reclaims_total`
- `scheduler_enqueues_total{schedule_id}`

## Docker, Startup, and Developer Commands
- `docker-compose.yml` services:
- `redis` with persistence.
- `api` on `:8000`.
- `worker` (default queue `default`).
- `scheduler` loading `config/schedules.yaml`.
- Shared volume for SQLite DB path `/app/data/mini_celery.db`.
- Commands:
- `make up` -> `docker compose up --build -d`
- `make down` -> `docker compose down -v`
- `make lint` -> `ruff check . && ruff format --check . && mypy mini_celery`
- `make test` -> compose-based test run with Redis + pytest.
- `make loadtest` -> CLI loadtest against running API.
- Fallback for no `make`: `python scripts/dev.py <up|down|lint|test|loadtest>`.

## Automated Tests
- Unit tests:
- State transition rules including invalid transitions rejection.
- Backoff/jitter bounds and monotonic growth under cap.
- API idempotency behavior for non-terminal duplicates.
- Worker idempotency short-circuit when prior success exists.
- Integration tests:
- At-least-once with crash: kill worker mid-job, second worker reclaims from PEL and completes.
- DLQ after max attempts: retryable task exceeds attempts, ends in `DEAD`, entry appears in `jobs:dlq`.
- Delayed execution: job with future `run_at` never runs early; runs after due time.
- Scheduler restart dedupe: same `(schedule_id, planned_time)` enqueued once across restart.
- `make test` exits non-zero on any failing unit/integration test.

## README Content Plan
- Architecture diagram and service responsibilities.
- Reliability walkthrough for worker crash, Redis restart, scheduler restart.
- Quickstart with exact commands.
- API examples for each endpoint.
- How to add a new task type (Pydantic payload + registry + tests).
- Design tradeoffs section.
- Assumptions section.

## Assumptions and Defaults
- UTC-only timestamps; RFC3339 accepted/emitted with timezone.
- Manual retry resets `attempts` to `0`, clears `last_error`, preserves `id`, `type`, `payload`, `idempotency_key`.
- Default queue name is `default`.
- Default `max_attempts=3`.
- Worker concurrency uses asyncio tasks, not multiprocessing.
- Backoff defaults: `base=1s`, `multiplier=2.0`, `cap=30s`, `jitter_ratio=0.2`.
- Visibility timeout default `30s`; reclaim interval default `5s`.
- SQLite runs in WAL mode with busy timeout to reduce write contention.
- Extra endpoint `GET /jobs/{job_id}/events` is added to support CLI tailing cleanly.
- Because this environment lacks local `docker` and `make`, implementation will include script fallbacks while preserving required `make` targets and Docker-first workflow.
