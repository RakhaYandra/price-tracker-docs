# Price Tracker — FSD (Functional Specification Document)

Per-feature behavior. Normative behavior lives here and in API-GUIDE.md;
IMPLEMENTATION.md says what to build first and how to prove it. Each
feature lists the FR IDs it implements (owned by SRS.md).

## FS-01 Auth and sessions (FR-012)

Behavior: email + password login creates a session. The access token
(opaque, Redis, 15-minute TTL) is held in frontend memory and sent as
`Authorization: Bearer`. The refresh token (opaque, Redis, 30-day TTL,
rotated on each use) travels in an httpOnly cookie scoped to
`/api/v1/auth` with `SameSite=Strict` and `Secure`. Logout revokes
both. "Logout all devices" revokes every session row for the user.

Inputs: `email`, `password`. Outputs: access token + refresh cookie.
On boot the web app calls `POST /auth/refresh` once; while it runs the
UI shows "restoring session".

Error cases: wrong credentials → single generic message ("invalid
email or password"), never "email not found". More than 5 attempts per
minute per IP → 429 with `Retry-After` and exponential lockout.
Expired access → 401 → client refreshes once → retries → 401 again
means logged out. Expired or reused refresh token → 401, all sessions
for that token family revoked (reuse signals theft).

Acceptance criteria: unauthenticated calls to any `/api/v1/*` except
login/refresh return 401; refresh rotation invalidates the old token;
lockout triggers on the 6th rapid attempt; every mutating request is
attributed to an actor in the audit trail.

## FS-02 Tracker create and validate (FR-001, FR-002)

Behavior: the user submits a URL plus optional custom name, target
price, and check interval. The API validates synchronously (no
scraping) and stores the tracking target.

Inputs: `url` (required), `name` (optional), `target_price_minor`
(optional), `currency`, `interval_seconds`. Outputs: tracker record
with `enabled=true`, `next_check_at = now + interval`.

Error cases: non-absolute or non-HTTPS URL → 400; unsupported host →
422 `unsupported-source` problem; interval below the 300-second floor
→ 422; non-positive target price → 422; duplicate URL → 409.

Acceptance criteria: invalid URL rejected; unsupported source rejected
with a stable problem type; valid tracker appears on the dashboard;
paused tracker is never scheduled.

## FS-03 Tracker lifecycle (FR-013)

Behavior: list (cursor-paginated, filterable by status and source),
get one, update name/target/interval, pause/resume, delete.
Pause sets `enabled=false` and clears `next_check_at`; resume
recomputes it. Delete removes the tracker and cascades jobs,
observations, rules, and deliveries.

Inputs: tracker ID plus the fields to change. Outputs: updated
record or 204.

Error cases: unknown ID → 404; update during a running check is
allowed (check uses a snapshot); delete is irreversible and requires
ConfirmDialog in the UI.

Acceptance criteria: paused trackers vanish from scheduling queries;
list pagination is stable under concurrent inserts (cursor, not
offset); deleted data is gone from every list endpoint.

## FS-04 Manual check (FR-003)

Behavior: `POST /trackers/{id}/checks` creates a check job with a
unique idempotency key and returns 202 immediately. The worker picks
it up from the Redis queue. The UI polls `GET /checks/{id}` with
`refetchInterval` until the status is terminal, then invalidates the
tracker queries.

Inputs: tracker ID. Outputs: `{check_id, status: pending}`.

Error cases: unknown tracker → 404; paused tracker → 409; a check
already running for the tracker returns the existing check ID (lease),
not a duplicate.

Acceptance criteria: the API responds without waiting for the scrape;
the user can see check status; a successful check creates exactly one
observation; a failed check exposes a safe error category.

## FS-05 Scheduled check via Redis queue (FR-004)

Behavior: every tick the scheduler finds trackers with
`next_check_at <= now` and `enabled=true`, claims each with a
per-tracker Redis lease, creates a `check_jobs` row and a Redis queue
entry, and sets `next_check_at = now + interval`. Only the leader
(scheduler lock holder) schedules. Workers consume `pt:q:check`,
bounded by configured concurrency. Transient failures retry with
backoff and jitter; exhausted jobs become `failed`. The reconciler
re-enqueues stale `pending`/`running` jobs so a Redis flush or worker
crash loses no work.

Inputs: scheduler tick, config (tick, concurrency, retry policy).
Outputs: job rows and queue entries; metrics (due count, enqueue
lag).

Error cases: lease already held → skip (another worker has it);
leader lock lost mid-tick → stop scheduling, keep serving nothing
stale; queue full → backpressure, jobs stay `pending` in PostgreSQL.

Acceptance criteria: a due tracker is checked within one interval;
duplicate ticks never create duplicate jobs; killing a worker
mid-check results in exactly-once re-execution semantics via the
reconciler, never in a lost job.

## FS-06 Source adapter and Blibli (FR-004, FR-010)

Behavior: each source implements the narrow adapter interface; the
domain only ever sees `ObservationData`. MVP ships the Blibli adapter
(`https://www.blibli.com/p/<slug>`, bare URL, no query string). The
adapter matches the URL, fetches with timeout/size/redirect policy,
parses with goquery, extracts name/price/availability, validates the
price (positive, sane magnitude), and returns normalized data
including `currency_exponent`.

Inputs: tracker URL. Outputs: `ObservationData` or a categorized
adapter error.

Error cases: non-matching URL is never attempted (registry rejects
it); ambiguous extraction → `extraction_failed`, never a guessed
price; unparsable availability → `unknown`; upstream 5xx →
`source_5xx` (retryable); 4xx → `source_4xx` (not retryable without a
state change).

Acceptance criteria: URL matching is exact per `host_pattern`; fixture
tests cover price, out-of-stock, and layout-drift cases with no live
network; no source-specific HTML structure leaks outside the adapter
package.

## FS-07 Observation persistence and money (FR-005, FR-006, FR-007)

Behavior: a successful check writes exactly one observation row in the
same transaction that updates the tracker (`last_checked_at`,
`next_check_at`) and the job state. Observations are append-only and
carry `price_minor`, `currency`, `currency_exponent`, `availability`,
`observed_at` (UTC), `duration_ms`, and `source_product_ref`.

Inputs: validated `ObservationData`. Outputs: observation row.

Error cases: observation write failure → the whole transaction rolls
back and the check is `failed`, never "successful check" with no data;
NULL price is allowed only with `availability != in_stock`; a failed
notification never invalidates a stored observation.

Acceptance criteria: observations immutable; analytics reproducible
from rows; money fields always present together; no float anywhere in
storage, API, or state.

## FS-08 Price history and analytics (FR-005)

Behavior: date-range queries over immutable observations return the
series plus derived metrics: current, lowest, highest, average,
absolute change, percentage change, lowest within the selected
period. Analytics are computed from observations, never stored
separately (except short-lived dashboard caches).

Inputs: tracker ID, `from`/`to`, cursor. Outputs: series page plus
metric block.

Error cases: empty range → empty series with zeroed metrics and an
empty-state UI, not an error; single observation → change metrics
report "no previous"; mixed currencies in range → metrics computed
per currency, never converted.

Acceptance criteria: metrics match a recompute from raw rows;
cursor pagination is stable; the chart and the table show the same
numbers.

## FS-09 Alert rules and dedup (FR-008, FR-009)

Behavior: after each successful observation the engine evaluates the
tracker's active rules: `target_price` (price <= threshold),
`price_drop_percent` (drop vs previous >= threshold), `back_in_stock`
(availability `out_of_stock|unknown` → `in_stock`). Rules have state
`inactive|active`. Transition `inactive → active` emits one event;
`active → active` emits nothing; `active → inactive` resets (and P1
adds the back-to-normal path).

Critical edge case: a rule evaluates only when
`observation.currency == rule.currency`. On mismatch the rule records
`skipped_currency_mismatch`, which is neither TRUE nor FALSE and never
resets state — a currency change must not cause a spurious reset or a
spurious fire.

Inputs: observation + active rules. Outputs: zero or one alert event
per rule per observation.

Acceptance criteria: FALSE → TRUE notifies exactly once; TRUE → TRUE
is silent; TRUE → FALSE resets; FALSE → TRUE notifies again; currency
mismatch never fires and never resets.

## FS-10 Telegram notification (FR-010)

Behavior: each alert event creates a notification job with an
idempotency key. The notification worker delivers via the Telegram Bot
API, retries transient failures with backoff, marks permanent
failures `failed`, and persists delivery status visible in operations.

Inputs: alert event (rule, tracker, price, threshold). Outputs:
delivery row with status and attempt count.

Error cases: Telegram 429 → respect `retry_after`; invalid chat/token
→ permanent failure, surfaced in settings with a "test" action; worker
crash mid-delivery → idempotency key prevents double-send on retry.

Acceptance criteria: notification job has an idempotency key;
transient failures retry; permanent failures become failed; delivery
status is visible in operations; one event never produces two
messages.

## FS-11 Check operations and diagnostics (FR-011)

Behavior: read APIs expose last check, duration, status, error
category, retry count, and source per tracker, plus recent failure
lists for the dashboard. Enough metadata to diagnose failures without
storing page content.

Inputs: tracker ID or global filters. Outputs: job rows and
aggregates.

Error cases: none user-facing; missing jobs return empty, not error.

Acceptance criteria: every failed check is explainable from stored
metadata; no secrets or page content in any operational read.

## FS-12 Dashboard (UI only)

Behavior: landing page showing active tracker count, trackers needing
attention (failed checks, target hits), recent price changes, recent
check failures, and target-price hits. Polls on a 30-second interval
via TanStack Query; manual refresh always available. Must show:
active count, attention list, recent changes, recent failures,
target hits.

Acceptance criteria: no blank page while loading (skeletons); empty
account shows a guided empty state; every number links to its detail.

## FS-13 Settings (UI + API)

Behavior: notification configuration (bot token, chat ID, test
action), default check interval, timezone for display, application
name. Server-validated; test-notification sends a real Telegram
message and reports success/failure.

Acceptance criteria: invalid token/chat fails the test action with a
clear message; timezone affects display only, never storage (UTC);
settings changes are audited to the actor.
