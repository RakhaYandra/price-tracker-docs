# Price Tracker — PRD (Product Requirements Document)

ID numbering in this document is shared with `SRS.md`, which owns the
numbering. `FSD.md` implements these IDs per feature (`FS-01..13`).

## 1. Overview / Executive Summary

Price Tracker monitors product prices from supported web sources, stores
historical observations, evaluates alert rules, and notifies the user when
meaningful price conditions occur.

Product statement:

> Track products once, let the system collect price history
> automatically, and notify when the price reaches a condition worth
> acting on.

Primary goal: build a general-purpose price monitoring system that
demonstrates scraping, scheduling, data processing, source adapters,
historical analysis, and alerting. Status: planned.

## 2. Problem Statement

Users monitor products manually because prices change over time.
Repeatedly opening product pages is inefficient, and a single current
price does not show whether a price is actually low relative to recent
history. Existing habit (bookmarks, memory) provides no history, no
alerts, and no record of failures when a page becomes unreadable.

## 3. Target Users

Primary user: a technical individual who wants to monitor products for
personal purchases. The user runs the system locally (Docker Compose),
adds product URLs, and receives Telegram notifications. One user account
exists in MVP; the authentication model must not hard-code single-user
assumptions (see NFR-008 in SRS.md).

## 4. Goals & Objectives

### MVP goals

- Add a product URL and validate that the source is supported (FR-001,
  FR-002).
- Fetch the product page manually and on a schedule (FR-003, FR-004).
- Extract product name, price, and availability; store immutable
  observations (FR-005, FR-006, FR-007).
- Display current and historical prices (FR-005).
- Configure target-price, drop-percentage, and back-in-stock rules;
  notify once per transition via Telegram without duplicates (FR-008,
  FR-009, FR-010).
- Record scraper and check failures with diagnosable metadata (FR-011).
- Require login for every state-changing and state-reading API call
  (FR-012).
- Provide a simple dashboard, operations view, and settings (FS-12,
  FS-13).

### Non-goals

- Full e-commerce marketplace; purchasing products; checkout automation.
- CAPTCHA bypass; authentication bypass; high-volume scraping;
  arbitrary proxy rotation.
- Public multi-tenant SaaS in MVP.
- Machine-learning price prediction.

## 5. Feature Requirements

| ID | Requirement | Maps to |
|---|---|---|
| FR-001 | The system shall create a tracker from a valid supported product URL. | FS-02 |
| FR-002 | The system shall validate URL scheme, host, source support, and configured limits. | FS-02 |
| FR-003 | The system shall allow an authenticated client to trigger a non-blocking product check. | FS-04 |
| FR-004 | The system shall execute due trackers according to their configured interval. | FS-05, FS-06 |
| FR-005 | The system shall store each successful observation as an immutable record. | FS-07, FS-08 |
| FR-006 | The system shall represent availability as `in_stock`, `out_of_stock`, or `unknown`. | FS-07 |
| FR-007 | The system shall store monetary values as integer minor units with an explicit currency exponent, not floating-point values. | FS-07 |
| FR-008 | The system shall evaluate active rules after a successful observation. | FS-09 |
| FR-009 | The system shall avoid duplicate notifications while a rule remains continuously true. | FS-09 |
| FR-010 | The system shall retry transient notification failures and persist final delivery status. | FS-10 |
| FR-011 | The system shall record enough metadata to diagnose check failures without storing unnecessary third-party page data. | FS-11 |
| FR-012 | The system shall require authentication for all API access and support session login, refresh, and logout. | FS-01 |
| FR-013 | The system shall support tracker lifecycle operations: list, get, update, pause/resume, and delete. | FS-03 |

## 6. User Stories

- AS-01: As a user, I want to add a product URL with a target price, so
  that I stop checking the page myself.
- AS-02: As a user, I want the system to reject unsupported URLs
  immediately, so that I know what can be tracked.
- AS-03: As a user, I want to trigger a manual check, so that I see a
  fresh price without waiting for the schedule.
- AS-04: As a user, I want price history with lowest/highest/average, so
  that I can judge whether the current price is actually good.
- AS-05: As a user, I want exactly one Telegram message when my target
  price is first reached, so that I am informed but not spammed.
- AS-06: As a user, I want to pause a tracker, so that seasonal products
  do not consume check budget.
- AS-07: As a user, I want to see why a check failed, so that I can tell
  a broken page from a broken system.
- AS-08: As a user, I want to log in, so that my trackers and
  notification settings are not exposed to anyone with network access.

## 7. Acceptance Criteria

| ID | Criterion | Covers |
|---|---|---|
| AC-01 | Adding a supported product URL creates a tracker visible on the dashboard. | FR-001 |
| AC-02 | An invalid or unsupported URL is rejected with a machine-readable problem type. | FR-002 |
| AC-03 | A manual check returns 202 without waiting for the scrape; the observation appears exactly once on success. | FR-003, FR-005 |
| AC-04 | A due tracker is checked automatically within one interval of becoming due. | FR-004 |
| AC-05 | Observations are never mutated after creation; history queries return them ordered by observed time. | FR-005 |
| AC-06 | Unparsable availability is stored as `unknown`, never guessed. | FR-006 |
| AC-07 | Money in API responses always carries `price_minor`, `currency`, and `currency_exponent`; no float fields exist. | FR-007 |
| AC-08 | When a rule transitions FALSE → TRUE, exactly one notification is created and delivered. | FR-008, FR-010 |
| AC-09 | While a rule stays TRUE, no further notifications are created; after FALSE → TRUE again, one notification is created. | FR-009 |
| AC-10 | A failed check exposes a safe error category and duration; no stack trace, SQL error, or upstream secret leaks. | FR-011 |
| AC-11 | Unauthenticated API calls return 401; login, refresh, and logout behave as specified in API-GUIDE.md. | FR-012 |
| AC-12 | Paused trackers are excluded from scheduling; deleted trackers and their jobs are gone from all lists. | FR-013 |

## 8. Scope & Constraints

In scope (P0): one real source (Blibli), manual check, scheduled check,
history, target-price alert, Telegram, login. P1: second adapter
(dummyjson), lowest-in-period alert, back-in-stock alert, better scraper
diagnostics. P2: source comparison, seller normalization, richer
notification channels, import/export. Explicitly deferred: prediction,
recommendation engine, automated purchasing, public SaaS multi-tenancy.

Hard constraints:

1. Respect source terms and applicable access policies. robots.txt is a
   crawl permission, not a terms-of-service grant. Marketplace terms
   generally forbid automated scraping; this project is scoped as
   personal, low-rate, non-commercial use.
2. Do not bypass CAPTCHA, authentication, robots/access controls, or
   anti-bot protections. Do not use proxy rotation.
3. Apply conservative request rates (per-host token bucket;
   `Crawl-delay` honored where published).
4. Avoid storing unnecessary page content; fixtures must be sanitized
   and minimal.
5. Do not assume a source's HTML structure is permanent; fail safely when
   extraction is ambiguous (record `unknown`, never guess).
6. Every feature must have a testable acceptance criterion.
7. Documentation is part of the implementation, not an afterthought.

## 9. Timeline / Milestones

| Milestone | Content | DoD |
|---|---|---|
| M0 | Repo setup, CI, Compose, contracts | `make test/lint/build` green |
| M1 | Database, domain, auth skeleton | migrations + pure domain tests green |
| M2 | Auth + Blibli adapter + manual check | login roundtrip, 1 fixture-passing adapter |
| M3 | Scheduler + queue + observation pipeline | due tracker checked within one interval |
| M4 | Alerts + Telegram + dashboard | one notification on first TRUE, dedup proven |
| M5 | Operations, settings, hardening | SSRF matrix + security tests green |
| M6 | P1 second adapter + polish | two adapters, contract generality demonstrated |

## 10. Dependencies

- Blibli product pages reachable and parseable; robots.txt `Allow`
  for the product URL shape (verified 2026-10-01).
- dummyjson.com availability for the P1 adapter and its fixtures.
- Telegram Bot API for the MVP notification channel.
- Docker images: PostgreSQL, Redis, `pandoc/latex:3.1` (docs only).
- Go toolchain and Node toolchain as documented in the code repos.

## 11. Success Metrics

MVP is successful when a user can:

1. Add a supported product.
2. Manually check it.
3. See a valid observation.
4. See historical observations after repeated checks.
5. Configure a target price.
6. Receive exactly one notification when the target condition first
   becomes true.
7. Recover from transient scraper failure without crashing the worker.
8. See why a check failed.

Operational metrics (not vanity metrics): successful check rate,
scraper failure rate by source, median and p95 check duration, alert
delivery success rate, duplicate alert rate, active tracker count,
observations per product.

## 12. Out of Scope

Full e-commerce marketplace; purchasing; checkout automation; CAPTCHA
or authentication bypass; high-volume scraping; proxy rotation; public
multi-tenant SaaS; ML price prediction; automated purchasing (never add
unless the scope is deliberately redesigned around its legal, security,
and operational risks).
