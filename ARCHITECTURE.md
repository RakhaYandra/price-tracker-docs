# Price Tracker — Architecture

Price Tracker uses a modular monolith with asynchronous workers. This is
intentional: the domain is small enough that microservices would add
deployment, networking, and observability complexity without meaningful
value. The Go backend is one codebase with three processes; the React
frontend is a separate repository.

## 1. System Overview

```text
+--------------+       HTTPS        +------------------+
|  React Web   | -----------------> |     Go API        |
|  (separate   |                    |  HTTP / App /     |
|   repo)      | <----------------- |  Domain layers    |
+--------------+     JSON           +--------+---------+
                                              |
                       +----------------------+----------------------+
                       |                      |                      |
                       v                      v                      v
                  PostgreSQL                Redis              Supported sources
                  (authority)          (queue/rate/locks/     (Blibli, dummyjson)
                                        sessions)
                       ^                      ^
                       |                      |
                +------+----------------------+------+
                |  Scheduler  /  Worker  /  Notifier |
                +-----------------------------------+
```

## 2. Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Language | Go (version pinned in `go.mod`) | team standard, stdlib strength |
| HTTP | Gin | team standard from Pulse; non-trivial route surface |
| Database | PostgreSQL | relational, consistency-sensitive history |
| Queue / rate / locks / sessions | Redis | queue semantics + token buckets need TTL and sorted sets (ADR-003) |
| Frontend | React 19 + Vite + TypeScript strict | product UI precedent (shiftbase/lifeos) |
| UI system | Tailwind CSS v4 + shadcn (Radix primitives) | proven twice; 17-component contract |
| Chart | Recharts 3.x (never 2.x — broken on React 19) | declarative, token-colored |
| Data fetching | TanStack Query | polling + invalidation + query-state mapping |
| Scraping | Go HTTP client + goquery, source adapters | narrow contract (ADR-004) |
| Notifications | Telegram first | personal project, small adapter |
| Local dev | Docker Compose | api + worker + scheduler + postgres + redis + web |
| API contract | OpenAPI 3.1.1 | language-agnostic interface description |
| API errors | RFC 9457 Problem Details | machine-readable errors (ADR-009) |
| Password hashing | Argon2id | memory-hard (ADR-012) |

## 3. Infrastructure

Development (`docker compose`): api, worker, scheduler, postgres,
redis, web. Production: containerized API, containerized worker,
containerized scheduler (single leader via Redis lock), managed
PostgreSQL if available, Redis required, HTTPS at ingress, secrets via
environment or secret manager. Redis is a hard dependency (ADR-003),
not an optional add-on.

## 4. Database Schema

PostgreSQL is the source of truth for trackers, observations, alerts,
job state, users, and sessions. Redis is transport and cache, never
authority. Core tables: `sources`, `trackers`, `price_observations`,
`check_jobs`, `alert_rules`, `notification_deliveries`, `users`,
`sessions`. Full DDL, constraints, and indexes: DATABASE-GUIDE.md.

Money: `price_minor BIGINT` + `currency CHAR(3)` +
`currency_exponent SMALLINT` (0 for IDR/JPY, 2 for USD). Time: always
`TIMESTAMPTZ`, stored UTC, converted only at presentation.

## 5. API Design

RESTful resources under `/api/v1`, JSON request/response, OpenAPI
3.1.1 as the source contract, RFC 9457 errors, cursor pagination for
histories, `Idempotency-Key` for mutations that enqueue work. Auth:
bearer access token in memory plus rotating refresh token in an
httpOnly cookie scoped to `/api/v1/auth`. Full reference:
API-GUIDE.md.

## 6. System Components

API process: authentication, CRUD, dashboard queries, manual-check
requests, alert configuration, operational reads. Must not perform
long-running scraping synchronously, send slow notifications inside the
request lifecycle, or contain source-specific scraping rules.

Scheduler: finds due trackers, claims them with a per-tracker lease,
enqueues check jobs to Redis, holds the singleton leader lock. Must
not scrape.

Worker: consumes check jobs, invokes the source adapter, normalizes
the result, persists the observation transactionally, evaluates alert
rules, enqueues notification jobs. A broken adapter must not crash the
worker process.

Notification worker: delivers notifications, retries transient
failures with backoff, persists delivery status. A failed notification
must not invalidate a stored observation.

Reconciler (inside scheduler): re-enqueues PostgreSQL jobs that are
`pending` and stale or `running` with a dead heartbeat, covering Redis
flushes and worker crashes.

## 7. Data Flow Diagram

```text
Scheduler
  |
  v
Due tracker (lease claimed)
  |
  v
Check job -> Redis queue (pt:q:check)
  |
  v
Worker -> Source adapter
  |
  v
Normalized ObservationData
  |
  v
PostgreSQL transaction (observation + tracker state + job state)
  |
  v
Alert evaluation (FALSE -> TRUE = emit event)
  |
  v
Notification job -> Redis queue (pt:q:notify)
  |
  v
Telegram
```

Auth flow:

```text
Login (email + password, rate-limited)
  |
  v
Access token (opaque, memory) + refresh token (opaque, httpOnly cookie)
  |
  v
API calls: Authorization: Bearer <access>
  |
  v
401 -> POST /auth/refresh (cookie) -> retry once -> 401 = logout
```

## 8. Security Considerations

Threat model summary: untrusted third-party HTML, user-supplied URLs
(SSRF), credential theft, token replay, secret leakage in logs,
dependency compromise, source ToS violation. Controls: source-host
allowlist, per-hop redirect revalidation, private-IP rejection,
response-size and timeout caps, Argon2id passwords, opaque rotating
tokens, rate-limited login, minimal logging (no secrets, no page
content). Full model: SECURITY-GUIDE.md.

## 9. Scalability Plan

MVP: 1 API + 1 worker + 1 scheduler (leader lock prevents duplicate
scheduling even with replicas). Later: API x N, worker x N, scheduler
x 1 (or leader-elected), Redis, PostgreSQL. Do not introduce
distributed coordination before it is needed.

## 10. Performance Targets

Normal API reads target p95 < 300 ms under the defined MVP load
profile, excluding intentionally asynchronous operations (scrapes,
notifications). Scheduler tick finds due trackers in bounded time via
the partial index on `next_check_at`. History queries paginate by
cursor on `(tracker_id, observed_at)`.

## 11. Error Handling & Logging

API errors use RFC 9457 `application/problem+json` with stable `type`
URIs; never expose stack traces, SQL errors, upstream secrets, or
internal network details. Check failures use a safe error category
enum (`timeout`, `response_too_large`, `redirect_rejected`,
`unsupported_content`, `extraction_failed`, `source_5xx`,
`network_error`). Logs include request/check ID, tracker ID, source,
operation, outcome, and duration. Logs never include tokens, secrets,
authorization headers, or page contents.

## 12. Third-party Integrations

Blibli (MVP source; robots.txt `Allow: /p/*$` verified 2026-10-01;
bare product URL only, no query string). dummyjson.com (P1 second
adapter; plain JSON API). Telegram Bot API (MVP notification channel).
All provider code sits behind narrow interfaces and is replaceable
without touching domain logic.

## 13. Deployment Strategy

Containers for api, worker, scheduler, web (static build). Environment
variables for all configuration (see section 16). Migrations run at
deploy before traffic. Health endpoints `/healthz` (liveness) and
`/readyz` (postgres + redis reachable). Rollback is forward-only
migration compatible or restore from backup.

## 14. Documentation Plan

This repo is the design source. Code repos carry only operational
docs: `price-tracker/README.md` (setup, env, supported sources,
troubleshooting), `api/openapi.yaml` (generated/validated contract),
`price-tracker-web/README.md` (dev, build, e2e). Guides here stay
normative; code comments stay minimal.

## 15. Concurrency

Worker concurrency is explicitly bounded by config. Per-source rate
limiting uses Redis token buckets (`pt:rl:<source>`). Per-tracker
leases (`pt:lock:trk:<id>`) prevent concurrent checks of the same
product. Scheduler leadership uses `pt:lock:sched` (SET NX PX). Job
execution records carry idempotency keys
(`check:{tracker_id}:{scheduled_bucket}`); workers tolerate duplicate
delivery.

## 16. Configuration

Precedence: CLI flags, then environment, then config file, then
defaults. Required variables: database URL, Redis URL, Telegram bot
token and chat ID, session TTLs, worker concurrency bounds,
per-source rate floors, scheduler tick, scraper timeout and size cap,
min check interval floor, web origin allowlist.
