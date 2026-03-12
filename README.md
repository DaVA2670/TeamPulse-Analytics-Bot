# Team Communication Analytics Bot

Backend-only Telegram analytics bot that ingests team chats, computes deterministic process signals (open loops, unresolved discussions, missing owners and deadlines, slow mention responses), and delivers daily/weekly fact-grounded reports to a whitelisted owner via private Telegram messages.


## 1. High-level Features

- **Webhook-first ingestion**: Receives Telegram updates via HTTPS webhook, normalizes them to internal DTOs, stores messages/edits/mentions in PostgreSQL with full lineage.
- **Deterministic analytics**:
  - Open loops (questions/tasks with no answer beyond threshold).
  - Unresolved long discussions.
  - Missing owner / missing deadline for task-like threads.
  - Slow responses to direct mentions (SLA on `@mentions`).
  - Basic thematic tagging of threads.
- **Fact-grounded reporting with LLM**:
  - Analyst → Verifier → Finalizer LLM pipeline using JSON-schema-validated structured outputs.
  - Verifier compares all claims against deterministic facts and rejects unsupported drafts.
  - Finalizer builds the final narrative only from supported claims.
- **Private, auditable delivery**:
  - Reports are sent **only** to whitelisted owner Telegram user IDs in private chats.
  - Nothing is ever posted into group/supergroup chats.
  - All important events are written to `audit_logs` (ingestion, analytics, reporting, retention, LLM failures).
- **Scheduling, retention, operations**:
  - Celery beat schedules daily/weekly report jobs and retention jobs.
  - Retention service cleans or anonymizes old raw data while keeping aggregates.
  - Health endpoints and structured logging for observability and safe restarts.

The system follows a strict constitution focused on determinism, idempotency, and auditability. LLMs are **only** used to turn verified facts into text, never to invent facts.

---

## 2. Architecture Overview

The project is fully backend-only and runs as a small Docker Compose stack.

### Components

- `api` (FastAPI + Uvicorn):
  - `/webhook/telegram` – Telegram webhook endpoint.
  - `/health/live` and `/health/ready` – liveness and readiness probes.
  - Optional admin endpoints (e.g. trigger report, diagnostics).
- `worker` (Celery worker):
  - Processes analytics, reporting, and maintenance tasks.
- `beat` (Celery beat):
  - Schedules daily/weekly reports and retention cleanup.
- `db` (PostgreSQL):
  - Primary source of truth for chats, messages, threads, signals, reports, LLM runs, audit logs, owner settings.
- `redis`:
  - Used as Celery broker and for transient locks/idempotency only (never as source of truth).

### Code Layout (short version)

- `src/app/` – app config, FastAPI factory, logging, container wiring, health endpoints.
- `src/api/` – HTTP routes (webhook, admin) and dependency helpers.
- `src/telegram_bot/` – aiogram bot wrapper, handlers, Telegram DTO models.
- `src/db/` – SQLAlchemy models, enums, base, repositories, Alembic migrations.
- `src/analytics/` – threading logic and deterministic analytics rules.
- `src/services/` – domain services (ingestion, threading, analytics, reporting, retention).
- `src/llm/` – LLM client wrapper, schemas, and analyst/verifier/finalizer stages.
- `src/reporting/` – report assembler, delivery (Telegram DM only), audit helpers.
- `src/workers/` – Celery app and task modules (analytics, reporting, maintenance).
- `src/security/` – authorization (owner whitelist) and privacy helpers (log redaction).
- `tests/` – unit, integration, and contract tests (webhook, analytics, LLM schemas, reporting, retention, retries).

---

## 3. Data Model (Summary)

Key tables (see `specs/001-team-communication-analytics/data-model.md` for details):

- `chats` – connected Telegram chats (group/supergroup) and their owners.
- `users` – Telegram users seen in connected chats.
- `messages` – current canonical state of each relevant message.
- `message_edits` – optional edit history (for deep audit).
- `mentions` – extracted mentions from messages.
- `threads` – logical discussion threads.
- `thread_participants` – participants per thread.
- `process_signals` – deterministic signals (open_loop, unresolved_thread, missing_owner, missing_deadline, slow_mention_response, theme_pattern).
- `reports` / `report_items` – report runs and included signal items.
- `llm_runs` – LLM calls with request/response payloads and statuses.
- `audit_logs` – important actions for operations and auditability.
- `owner_settings` – thresholds (open loop, slow response, unresolved rules), retention windows, time zone.

All mutable entities have status and timestamps; critical entities have idempotency keys or uniqueness constraints to enforce safe retries.

---

## 4. How to Run Locally (Quickstart)

This is a condensed version; for full details see `specs/001-team-communication-analytics/quickstart.md`.

### 4.1 Prerequisites

- Docker and Docker Compose installed.
- Telegram bot token (create via BotFather).
- OpenAI-compatible API key (for LLM; can be mocked during development).
- PostgreSQL and Redis images available (will be pulled automatically by Compose).

### 4.2 Configure Environment

1. Clone the repository.
2. Create `.env` in the repository root (or copy from `.env.example` if present) and set at minimum:
   - `BOT_TOKEN` – Telegram bot token.
   - `POSTGRES_DSN` – DSN for PostgreSQL used by the stack.
   - `REDIS_URL` – Redis URL for Celery.
   - `LLM_API_KEY` – key for the LLM provider.
   - `OWNER_TELEGRAM_USER_ID` – Telegram user id of the owner who will receive reports.
3. Optionally tweak defaults:
   - `OPEN_LOOP_THRESHOLD_HOURS` (default 24).
   - `SLOW_RESPONSE_THRESHOLD_HOURS` (default 4).
   - `RETENTION_DAYS` for raw messages (default 180).

### 4.3 Start the Stack

From the repository root:

```bash
docker compose up --build
```

This will start:

- `api` – FastAPI app with webhook and health checks.
- `worker` – Celery worker.
- `beat` – Celery beat scheduler.
- `db` – PostgreSQL.
- `redis` – Redis.

Apply Alembic migrations automatically on startup or manually inside the `api` container:

```bash
alembic upgrade head
```

### 4.4 Configure Telegram Webhook

1. Expose `api` to the internet using `ngrok` or a public domain.
2. Call Telegram `setWebhook` with:
   - URL: `https://<public-host>/webhook/telegram`
   - Secret token (if `TELEGRAM_WEBHOOK_SECRET` is configured).

### 4.5 Connect a Test Chat

1. Create a group or supergroup chat in Telegram.
2. Add the bot to the group and give it permission to read messages.
3. Send a few test messages (normal, replies, edits, `@mentions`) and confirm (via logs or admin endpoint) that updates are ingested and stored.

### 4.6 Generate Activity Scenarios

In the test chat, create representative scenarios:

- Question or task message that nobody answers for > threshold → should appear as **open loop**.
- Long discussion without clear resolution markers → **unresolved thread**.
- Task-like message with no clear owner → **missing owner**.
- Task-like message with owner but no deadline → **missing deadline**.
- Message with direct `@mention` where the mentioned user answers only after a long delay → **slow mention response**.

Wait until after the reporting window closes (or trigger report generation via an admin endpoint or direct Celery task call in development).

### 4.7 Validate Reports

Check that:

- The owner receives a private message containing the daily/weekly report.
- The report references your test scenarios (open loop, unresolved thread, missing owner/deadline, slow mention).
- No report is posted into any group or public chat.

### 4.8 Validate Idempotency, Retention, and Failures

- **Idempotency**: Re-send a captured Telegram update; confirm that no duplicate messages/signals/reports appear.
- **Retention**: Temporarily lower retention settings, run the retention Celery task, and verify that old raw messages/edits are cleaned or anonymized while aggregates remain consistent.
- **LLM failures**: Simulate LLM unavailability (e.g., invalid key / disabled network) and ensure reports are marked failed / rejected and not delivered, with audit logs for `llm_*_failed` events.

---

## 5. Configuration Reference (Core Settings)

Pydantic settings live in `src/app/config.py`. Key fields (env variables):

- `DATABASE_URL` / `POSTGRES_DSN` – PostgreSQL connection.
- `REDIS_URL` – Redis connection.
- `TELEGRAM_BOT_TOKEN` / `BOT_TOKEN` – bot token.
- `TELEGRAM_WEBHOOK_SECRET` – optional Telegram webhook secret.
- `LLM_API_KEY` – API key for OpenAI-compatible LLM.
- `OWNER_TELEGRAM_USER_IDS` – list of allowed owner IDs.
- Threshold defaults:
  - `OPEN_LOOP_THRESHOLD_HOURS` – default 24.
  - `SLOW_RESPONSE_THRESHOLD_HOURS` – default 4.
  - `RETENTION_DAYS` – default 180 for raw messages.
- HTTP client limits/timeouts for outbound calls (LLM, Telegram).

Owner-specific overrides (e.g. per-team thresholds, retention, time zone) are stored in `owner_settings` rows.

---

## 6. Development & Testing

### 6.1 Running Tests

From the repository root:

```bash
pytest
```

Test structure:

- `tests/unit/` – unit tests for analytics rules, threading, LLM verifier, prioritization, etc.
- `tests/integration/` – integration tests for webhook ingestion, analytics pipeline, report assembly, LLM pipeline, scheduling, retention, idempotent tasks, health checks, retries.
- `tests/contract/` – contract tests for Telegram webhook JSON and LLM schemas.

### 6.2 Local Development Notes

- All external calls (Telegram, DB, Redis, LLM) use bounded timeouts and retries.
- Celery tasks are idempotent; re-running analytics/reporting should not create duplicates.
- For local work you can monkeypatch `LLMClient` to avoid real HTTP calls (see existing tests in `tests/integration/test_llm_pipeline.py`).

---

## 7. Security, Privacy, and Compliance

- All secrets are loaded from environment variables; no tokens should be committed to the repo.
- Reports are delivered only to owner private chats; they are never posted to team channels.
- Logs are structured and avoid dumping full message content where not needed; additional redaction helpers are available in `src/security/privacy.py`.
- Data retention policies are enforced by code and can be tuned per owner.

For deeper non-functional requirements and guarantees, see `specs/001-team-communication-analytics/spec.md` and `.specify/memory/constitution.md`.

---

## 8. Russian README

For a full Russian-language overview and setup guide, see `README.ru.md` in the repository root.
