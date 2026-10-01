# Price Tracker — API Guide

RESTful JSON API under `/api/v1`. OpenAPI 3.1.1 (`api/openapi.yaml`
in the code repo) is the source contract; this guide is the readable
reference. Errors are RFC 9457 problems. Money is always the flat
triple `price_minor` / `currency` / `currency_exponent`.

## A. Overview

Base URL `/api/v1`. JSON request/response. ISO 8601 UTC timestamps.
Cursor pagination for histories and operational lists. Mutations that
enqueue work accept `Idempotency-Key` and return the same logical
result for duplicates within the retention window. Every endpoint
except login/refresh and health requires `Authorization: Bearer
<access-token>`; see section F.

## B. Design principles

- RESTful resource naming, plural collections, nouns not verbs
  (except `checks`, which are jobs, and `auth`, which is a session
  controller).
- JSON everywhere; `application/problem+json` for errors.
- Explicit pagination; no unbounded lists.
- Integer money; explicit currency and exponent.
- 202 (not 200) for accepted async work; the client polls job status.
- Stable problem `type` URIs the frontend can switch on.

## C. Endpoints

Auth:

```text
POST /api/v1/auth/login     {email, password} -> access token + refresh cookie
POST /api/v1/auth/refresh   (cookie) -> rotated tokens
POST /api/v1/auth/logout    revoke current session
GET  /api/v1/auth/me        current user
```

Trackers:

```text
POST   /api/v1/trackers
GET    /api/v1/trackers?status=&source=&limit=&cursor=
GET    /api/v1/trackers/{trackerId}
PATCH  /api/v1/trackers/{trackerId}
DELETE /api/v1/trackers/{trackerId}
POST   /api/v1/trackers/{trackerId}/pause
POST   /api/v1/trackers/{trackerId}/resume
```

Checks and observations:

```text
POST /api/v1/trackers/{trackerId}/checks   -> 202 {check_id, status}
GET  /api/v1/checks/{checkId}
GET  /api/v1/trackers/{trackerId}/observations?from=&to=&limit=&cursor=
```

Alerts, notifications, operations, settings:

```text
POST   /api/v1/trackers/{trackerId}/alerts
GET    /api/v1/trackers/{trackerId}/alerts
PATCH  /api/v1/alerts/{alertId}
DELETE /api/v1/alerts/{alertId}
GET    /api/v1/notifications?limit=&cursor=
GET    /api/v1/trackers/{trackerId}/checks?limit=&cursor=
GET    /api/v1/operations/failures?limit=&cursor=
GET    /api/v1/settings
PATCH  /api/v1/settings
POST   /api/v1/settings/test-notification
```

Health (unauthenticated):

```text
GET /healthz   liveness
GET /readyz    postgres + redis reachable
```

Create tracker:

```http
POST /api/v1/trackers
Content-Type: application/json
```

```json
{
  "url": "https://www.blibli.com/p/sample-product",
  "name": "Headphones X",
  "target_price_minor": 1100000,
  "target_currency": "IDR",
  "target_currency_exponent": 0,
  "interval_seconds": 21600
}
```

Response `201 Created` echoes the stored tracker including its
`currency`, `currency_exponent`, `enabled`, and `next_check_at`.

Manual check returns `202 Accepted` with `{check_id, status}` and
never waits for the scrape. Observations list returns `{data,
pagination: {next_cursor}}` with each row carrying `price_minor`,
`currency`, `currency_exponent`, `availability`, `observed_at`,
`duration_ms`.

## D. Common patterns

Cursor pagination: opaque `cursor`, `limit` with server cap; stable
under concurrent inserts. Problem mapping: the web client converts
`fieldErrors` to inline form errors and global failures to toasts
(see DESIGN-SYSTEM section 6). Polling: `GET /checks/{id}` with
`refetchInterval` until terminal, then invalidate tracker queries.
Partial update via PATCH with only changed fields; pause/resume are
explicit POST actions, not boolean flips.

OpenAPI fragment (create tracker, abbreviated):

```yaml
paths:
  /trackers:
    post:
      operationId: createTracker
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [url, interval_seconds]
              properties:
                url: { type: string, format: uri }
                name: { type: string, maxLength: 200 }
                target_price_minor: { type: integer, minimum: 1 }
                target_currency: { type: string, minLength: 3, maxLength: 3 }
                target_currency_exponent: { type: integer, minimum: 0, maximum: 4 }
                interval_seconds: { type: integer, minimum: 300 }
      responses:
        '201':
          description: tracker created
        '422':
          description: validation failure
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/Problem' }
```

## E. Versioning

URL versioning (`/api/v1`). Breaking changes require a new version;
additive changes (new fields, new endpoints) do not. `fieldErrors`
shape and problem `type` URIs are part of the versioned contract —
renaming a problem type is breaking.

## F. Authentication

Login is rate-limited (5/min/IP, then exponential lockout).
Successful login returns the access token in the body and sets the
refresh cookie (`HttpOnly; Secure; SameSite=Strict;
Path=/api/v1/auth`). The SPA keeps the access token in memory and
sends `Authorization: Bearer`. On 401 the client calls refresh once
and retries; a second 401 means logged out. Refresh reuse revokes the
token family. `RequireAuth` middleware guards everything except
login, refresh, `/healthz`, `/readyz`, `/metrics`. Authorization is
structurally present (deny-by-default) even though MVP has one
account.

## G. Error handling

Content type `application/problem+json`. Example:

```json
{
  "type": "https://price-tracker.local/problems/unsupported-source",
  "title": "Unsupported source",
  "status": 422,
  "detail": "The provided URL does not belong to a supported source.",
  "instance": "/api/v1/trackers",
  "fieldErrors": { "url": "use a Blibli product URL" }
}
```

Stable problem types: `validation-failed`, `unsupported-source`,
`duplicate-url`, `tracker-paused`, `check-in-progress`,
`rate-limited`, `unauthenticated`, `forbidden`, `not-found`,
`conflict`, `upstream-unavailable`, `internal`. Never expose stack
traces, SQL errors, upstream secrets, or internal network details.

HTTP status rules: 200 OK, 201 Created, 202 Accepted, 204 No Content,
400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found,
409 Conflict, 422 Unprocessable Content, 429 Too Many Requests
(with `Retry-After`), 500 Internal Server Error, 502 Bad Gateway,
503 Service Unavailable.

## H. Rate limiting & quotas

Per-IP login bucket (section F). Per-source scrape buckets in Redis
honor conservative floors above any published `Crawl-delay`.
Authenticated API reads share a generous per-token bucket; 429
responses carry `Retry-After` and a `rate-limited` problem body.
Minimum check interval floor is 300 seconds per tracker regardless
of plan.

## I. Webhooks

Not in MVP. Reserved: alert webhooks would be POST to a user URL with
HMAC signatures and the same idempotency keys as Telegram jobs.
Do not design it further until P2.

## J. SDK / client libraries

One typed client in the web repo (`src/lib/api.ts`): fetch wrapper,
bearer injection, single refresh-and-retry, problem parsing to
`ApiProblem` with `fieldErrors`. No other clients in MVP. Contract
tests (T5) validate real responses against `api/openapi.yaml` in CI.

## K. Changelog

API changes are recorded here per release: added endpoints, new
problem types, changed validation bounds, deprecations. v0.1.0 is the
baseline; the changelog starts empty and grows with the code.
