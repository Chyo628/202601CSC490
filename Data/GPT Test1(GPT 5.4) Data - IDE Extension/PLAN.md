# Mini Celery Repository Plan

## Summary
- Build a greenfield `src/miniq/` repository that cleanly separates API, broker, persistence, worker, scheduler, observability, CLI, and tests.
- Use Python 3.12, FastAPI/Uvicorn, SQLAlchemy 2.x with SQLite, Redis Streams with consumer groups, pytest, ruff, and mypy.
- Worker concurrency will be `asyncio` task based with a configurable `WORKER_CONCURRENCY`; task handlers are async, and `cpu_burn` is offloaded with `asyncio.to_thread` so the worker loop stays responsive.

## Key Changes
- Repository shape: `src/miniq/api`, `broker`, `db`, `worker`, `scheduler`, `tasks`, `observability`, `cli`, `tests`, plus `pyproject.toml`, `Dockerfile`, `docker-compose.yml`, `Makefile`, `config/schedules.yaml`, and `README.md`.
- Persistence: define `jobs`, `job_events`, and `schedule_ticks`. Extend `jobs` with `queue`, `cancel_requested`, `broker_enqueued_at`, `last_stream_id`, `schedule_id`, `finished_at`, and `dlq_published_at`. Use SQLAlchemy models, create tables on startup, and enable SQLite WAL plus busy timeout for multi-process/container access.
- State machine: allow `QUEUED->RUNNING`, `RUNNING->SUCCEEDED`, `RUNNING->FAILED`, `FAILED->QUEUED`, `RUNNING->CANCELED`, `QUEUED->CANCELED`, `RUNNING->DEAD`, and `DEAD->QUEUED`. A cancel request for a `RUNNING` job sets `cancel_requested=true` and appends a non-terminal event; the task must observe that flag and transition to `CANCELED` cooperatively.
- Broker abstraction: use one Redis stream per queue, named `jobs:{queue}`, and a consumer group named `workers`. Messages carry `job_id`, `queue`, `type`, and `enqueued_at`. `enqueue()` uses `XADD`, `read()` uses `XREADGROUP`, reclaim uses `XAUTOCLAIM` or `XCLAIM` after `visibility_timeout_ms`, and `XACK` happens only after the durable DB write for the final or next durable state succeeds.
- Duplicate message safety: worker execution starts with a compare-and-set `QUEUED->RUNNING` update. If a duplicate stream message arrives for a job already `RUNNING`, `SUCCEEDED`, `CANCELED`, or `DEAD`, the worker skips execution and only acknowledges when appropriate. This prevents concurrent duplicate execution even if a job is enqueued twice.
- Enqueue durability: API and scheduler always persist the job row first and then try `XADD`. `broker_enqueued_at IS NULL` is the recovery signal. Scheduler continuously backstops all due `QUEUED` jobs with `next_run_at <= now` and no broker marker so a crash between DB commit and `XADD` cannot strand a job.
- Retry policy: on handler error, append `RUNNING->FAILED`, compute deterministic exponential backoff with stable hash-based jitter derived from `(job_id, attempt)`, then append `FAILED->QUEUED` with `next_run_at=now+delay` if attempts remain. If attempts are exhausted, append `RUNNING->DEAD`, publish to `jobs:dlq`, set `dlq_published_at`, and then acknowledge the original stream entry.
- Task registry: ship `sleep`, `http_get`, `cpu_burn`, `fail_then_succeed`, and `pipeline_demo`. Each task has a Pydantic v2 payload model and per-task retry config. `pipeline_demo` is multi-step and checks `cancel_requested` between steps. `http_get` uses `httpx` with strict timeouts. `fail_then_succeed` succeeds once `attempts > failures_before_success`.
- API: implement the required endpoints exactly and add `GET /events?job_id=&cursor=&limit=` for CLI polling. `GET /jobs` returns cursor pagination ordered by `(created_at DESC, id DESC)` with an opaque base64 JSON cursor. `POST /jobs/{job_id}/retry` resets `attempts` to `0`, clears `last_error` and `result`, sets `state=QUEUED`, clears `cancel_requested`, and re-enqueues immediately if due.
- Scheduler: load recurring schedules from YAML with fields `id`, `cron`, `queue`, `type`, `payload`, and `max_attempts`. Store each planned fire in `schedule_ticks` with unique `(schedule_id, planned_time)`; only the insert winner may create and enqueue the job. On restart, compute due ticks from the last stored `planned_time` to `now`, so the same tick is never enqueued twice.
- Observability: use shared structured JSON logging with fields `service`, `job_id`, `stream_id`, `consumer`, `attempt`, `state`, and `schedule_id`. API exposes `/metrics`; worker and scheduler also expose Prometheus metrics on dedicated ports so `worker_reclaims_total` and `scheduler_enqueues_total` are scrapeable. `queue_depth` is computed from DB counts of due `QUEUED` jobs per queue, not raw stream length.
- CLI: use Typer for `miniq submit`, `status`, `cancel`, `retry`, `events-tail`, and `loadtest`. `loadtest` submits N jobs concurrently with `httpx.AsyncClient`, polls until terminal states, and reports throughput plus p50/p95/p99 submission and completion latency.

## Test Plan
- Unit tests: pure state-machine tests for allowed and forbidden transitions, cancel-request behavior, deterministic retry delay math and jitter bounds, and idempotency behavior for duplicate active submissions plus redelivery after success.
- Integration tests: start ephemeral Redis and a temp SQLite DB, then verify worker crash recovery by killing a worker mid-task before `XACK`, waiting past the visibility timeout, starting a new worker, and asserting reclaim plus successful completion. Verify DLQ behavior after max attempts. Verify delayed jobs do not run before `run_at`. Verify scheduler restart does not duplicate a cron tick by running the scheduler twice against the same planned time and asserting one `schedule_ticks` row and one job.
- Tooling: `make test` runs `pytest`; `make lint` runs `ruff check`, `ruff format --check`, and `mypy`; `make up` and `make down` wrap `docker compose up -d --build` and `docker compose down -v`; `make loadtest` runs the CLI against the compose stack.
- Compose: `docker-compose.yml` starts `redis`, `api`, `worker`, and `scheduler`, shares a named volume for the SQLite file, wires health checks, and passes env vars for Redis URL, DB path, visibility timeout, metrics ports, and worker concurrency.

## Assumptions
- The workspace is empty, so the deliverable is a fresh repository under `src/`.
- Submission-time idempotency only deduplicates non-terminal jobs, matching the prompt exactly; a new submission with the same key after a terminal state creates a new job.
- `run_at <= now` is treated as immediate enqueue.
- Manual retry resets `attempts` to `0`; this must be documented in `README.md`.
- Cron schedules do not backfill before the first scheduler startup; once a schedule has emitted at least one recorded tick, restarts do catch up missed ticks from the last stored planned time to current time.
- Integration tests require Docker or another compatible container runtime for Redis; the README should call that out as a prerequisite.
