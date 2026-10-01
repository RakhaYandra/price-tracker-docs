# Price Tracker — SRS (Software Requirements Specification)

Abridged IEEE 830. FR numbers are owned here; PRD section 5 and FSD
reference the same IDs. FR-001..FR-011 are unchanged since the draft;
FR-012 and FR-013 were appended for authentication and lifecycle.
NFR-008 changed from conditional to mandatory when full login entered
scope; NFR-011 and NFR-012 are new.

## 1. Introduction

Purpose: specify what Price Tracker must do and how well, as the
contract between requirements (PRD), behavior (FSD), and verification
(section 8). Scope: MVP single-real-source price monitoring with
scheduled checks, immutable history, alert rules, Telegram
notification, and login. Out of scope: purchasing, prediction,
multi-tenant SaaS. Definitions: tracker (a monitored product URL),
observation (one immutable price reading), check (one fetch attempt),
rule (an alert condition with inactive/active state), adapter (a
source-specific HTML/JSON reader behind a narrow interface).

## 2. Overall description

Product perspective: a modular monolith (Go API + scheduler + worker +
notifier) with PostgreSQL as authority, Redis as queue/rate/lock/
session transport, and a React SPA. User classes: the owner (full
control) — MVP has one account, but authorization checks must be
structurally present. Operating environment: Docker Compose locally;
containerized processes in production with HTTPS at ingress.
Constraints: personal low-rate non-commercial scraping only; no
CAPTCHA/auth bypass; no proxy rotation; conservative rates; minimal
page-content retention. Assumptions: Blibli product URL shape stays
fetchable; Telegram Bot API stays available; Redis is always up (hard
dependency, see NFR-012).

## 3. Specific requirements

### FR-001 Product creation
The system shall create a tracker from a valid supported product URL.

### FR-002 Product validation
The system shall validate URL scheme, host, source support, and
configured limits.

### FR-003 Manual checks
The system shall allow an authenticated client to trigger a
non-blocking product check.

### FR-004 Scheduled checks
The system shall execute due trackers according to their configured
interval.

### FR-005 Observation persistence
The system shall store each successful observation as an immutable
record.

### FR-006 Availability
The system shall represent availability as `in_stock`,
`out_of_stock`, or `unknown`.

### FR-007 Price normalization
The system shall store monetary values as integer minor units with an
explicit currency exponent, not floating-point values.

### FR-008 Alert evaluation
The system shall evaluate active rules after a successful
observation.

### FR-009 Alert deduplication
The system shall avoid duplicate notifications while a rule remains
continuously true.

### FR-010 Notification delivery
The system shall retry transient notification failures and persist
final delivery status.

### FR-011 Check diagnostics
The system shall record enough metadata to diagnose check failures
without storing unnecessary third-party page data.

### FR-012 Authentication and sessions
The system shall require authentication for all API access and support
login, token refresh with rotation, logout, and logout-all-devices.

### FR-013 Tracker lifecycle
The system shall support listing, retrieving, updating, pausing,
resuming, and deleting trackers with correct scheduling side effects.

## 4. System features

Features FS-01..FS-13 (see FSD.md) implement the requirements above:
FS-01 implements FR-012; FS-02 implements FR-001 and FR-002; FS-03
implements FR-013; FS-04 implements FR-003; FS-05 implements FR-004;
FS-06 implements FR-004 and FR-010; FS-07 implements FR-005, FR-006,
and FR-007; FS-08 implements FR-005; FS-09 implements FR-008 and
FR-009; FS-10 implements FR-010; FS-11 implements FR-011; FS-12 and
FS-13 are UI-only with API support from FR-001..FR-013.

## 5. Other nonfunctional requirements

### NFR-001 Availability
The API should remain available while individual scrapers fail.

### NFR-002 Reliability
A scraper exception must be isolated from the worker process.

### NFR-003 Performance
Normal API read operations should target p95 < 300 ms under the
defined MVP load profile, excluding intentionally asynchronous
operations.

### NFR-004 Large response safety
Scraper fetches must enforce response-size limits.

### NFR-005 Timeout
Every outbound source request must have a finite deadline.

### NFR-006 Concurrency
Worker concurrency must be explicitly bounded.

### NFR-007 Security
The application shall apply OWASP API security principles and SSRF
defenses.

### NFR-008 Auditability
State-changing operations must be attributable to an actor. (Now
mandatory: every request carries an authenticated identity.)

### NFR-009 Maintainability
Source-specific parsing shall be isolated from domain logic.

### NFR-010 Testability
Core domain rules must be testable without a live website.

### NFR-011 Authentication security
Passwords must use Argon2id; tokens must be opaque, random, and
revocable; login must be rate-limited; sessions must expire.

### NFR-012 Redis availability and recovery
Redis is a hard dependency. The system must survive a Redis flush or
worker crash without losing jobs, via PostgreSQL-authoritative state
and the reconciler.

## 6. Appendices

Reliability rules: retry transient network errors; no retry for
deterministic extraction failures without a state change; exponential
backoff with jitter; job idempotency; graceful shutdown; worker
cancellation via context. Data rules: timestamps in UTC; currency
stored explicitly with exponent; prices as integer minor units;
observations immutable; indexes for tracker/time queries. Operational
rules: logs include request/check ID, tracker ID, source, operation,
outcome, duration. Logs never include tokens, secrets, authorization
headers, or page contents. Compatibility: Go version pinned in
`go.mod`; PostgreSQL version documented and tested; browsers: current
major Chromium, Firefox, Safari.

## 7. Acceptance criteria

AC-01..AC-12 as listed in PRD.md section 7, each mapped to its FR.
MVP is done when all twelve hold plus the Definition of Done in
IMPLEMENTATION.md.

## 8. Traceability matrix

Test tiers: T1 unit domain (no I/O); T2 repository (testcontainers
PostgreSQL); T3 adapter fixture (no network); T4 integration
(API + PostgreSQL + Redis); T5 OpenAPI contract validation;
T6 web unit (vitest: formatters, query keys); T7 web e2e (Playwright);
T8 security (SSRF matrix, authz, rate limits); T9 performance
(benchmarks, k6).

| FS | FR | Tiers |
|---|---|---|
| FS-01 auth | FR-012 | T1, T4, T7, T8 |
| FS-02 create | FR-001, FR-002 | T1, T4, T5, T7 |
| FS-03 lifecycle | FR-013 | T2, T4, T5 |
| FS-04 manual check | FR-003 | T1, T4, T5, T7 |
| FS-05 scheduled check | FR-004 | T2, T4, T8 |
| FS-06 adapter | FR-004, FR-010 | T1, T3 |
| FS-07 persistence/money | FR-005, FR-006, FR-007 | T1, T2, T6 |
| FS-08 history/analytics | FR-005 | T2, T4, T6, T7 |
| FS-09 alerts/dedup | FR-008, FR-009 | T1, T2, T4 |
| FS-10 telegram | FR-010 | T1, T4, T8 |
| FS-11 operations | FR-011 | T4, T5 |
| FS-12 dashboard | (UI) | T6, T7 |
| FS-13 settings | (UI) | T4, T7 |
| NFR-003 | — | T9 |
| NFR-007 | — | T8 |
| NFR-011 | — | T4, T8 |
| NFR-012 | — | T4 |

Every FR appears in at least one FS row and every FS row maps to at
least one tier. UI-only features map to web tiers only.
