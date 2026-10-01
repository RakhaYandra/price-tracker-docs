# Price Tracker — ADR (Architecture Decision Records)

Single file, one record per decision. Status values: Accepted,
Proposed, Deprecated, Superseded. ADR-003 and ADR-007 were revised in
docs v0.1.0 before any code existed; the revision is recorded inline
rather than via supersession because nothing was implemented yet.

## ADR-001: modular monolith over microservices

Status: Accepted.

Context: need an API, a scheduler, a check worker, and a notification
worker for a personal-scale product. Alternatives: microservices per
component, serverless functions, single process with goroutines.
Factors: operational complexity, local setup cost, portfolio signal,
team familiarity.

Decision: a modular monolith — one Go codebase, three processes
(api, worker, scheduler), shared domain modules, PostgreSQL plus
Redis.

Consequences: positive — simpler development, one local Compose
stack, no network boundaries to debug; negative — a bad module can
still tangle imports if the dependency rule is not enforced;
cost — a lint rule and discipline; long-term — extraction remains
possible if `httpapi -> app -> domain` is respected.

Alternatives considered: microservices (why not: deployment,
networking, and observability complexity with zero domain payoff at
this scale); single process (why not: scheduler leadership and worker
scaling want separate lifecycles).

Related ADRs: ADR-002 (PostgreSQL), ADR-003 (Redis), ADR-014 (repo
split).

## ADR-002: PostgreSQL as source of truth

Status: Accepted.

Context: trackers, observations, alert state, job state, users, and
sessions need relational, queryable, consistency-sensitive storage.
Alternatives: SQLite (single file), MySQL, NoSQL document store.
Factors: transaction needs, time-series query patterns, hosted options,
team familiarity.

Decision: PostgreSQL is authoritative for all of the above. Redis is
never authoritative.

Consequences: positive — ACID transactions for check pipelines, rich
indexing for history queries; negative — migrations required, a
server to run; cost — Compose includes it, managed Postgres later;
long-term — time-partitioning possible if observations grow.

Alternatives considered: SQLite (why not: concurrent workers and a
separate API process outgrow file locking; also loses managed-hosting
path); document store (why not: the data is relational and benefits
from CHECK constraints and partial indexes).

Related ADRs: ADR-003, ADR-006, ADR-007.

## ADR-003: Redis as queue, rate limiter, lock, and session store

Status: Accepted (revised in docs v0.1.0; prior draft said Redis was
optional).

Context: checks must be queued with delay/retry/backoff semantics,
per-source request rates must be capped, the scheduler needs a leader
lock and per-tracker leases, and sessions need TTL storage.
Alternatives: PostgreSQL-backed queue with `FOR UPDATE SKIP LOCKED`
polling, in-process rate limiters, JWT sessions. Factors: TTL and
sorted-set ergonomics, polling cost, revocation needs, the fact that
PostgreSQL stays the state authority either way.

Decision: Redis is a hard MVP dependency for transport and
coordination: `pt:q:*` queues, `pt:rl:*` token buckets, `pt:lock:*`
locks/leases, `pt:tok:*`/`pt:rft:*` session tokens. PostgreSQL
`check_jobs` remains the authority over job state; a reconciler
re-enqueues stale jobs so a Redis flush loses no work.

Consequences: positive — delay/retry/DLQ and token buckets without
polling, sessions revocable instantly; negative — Redis is now a hard
dependency with its own availability story, and the portfolio contrast
with Pulse (also Redis-heavy) shrinks; cost — one more container and
a reconciler; long-term — swapping transports later is possible
because domain code only sees port interfaces.

Alternatives considered: PostgreSQL-backed queue only (why not:
polling loop, no clean token-bucket or TTL semantics, worse fit for
per-host rate limiting); in-process limits (why not: die with worker
restarts, do not survive scaling to N workers).

Related ADRs: ADR-002, ADR-005, ADR-011.

## ADR-004: adapter pattern for sources

Status: Accepted.

Context: each supported site structures its pages differently, and
structures change without notice. Alternatives: per-site code inline
in the worker, a generic CSS-selector config per source, a headless
browser. Factors: blast radius of a site redesign, testability,
domain purity.

Decision: each source implements a narrow adapter interface
(`Match(url) bool`, `Fetch(ctx, target) (ObservationData, error)`).
`ObservationData` is source-independent. The domain never sees HTML.

Consequences: positive — adapters independently testable with
fixtures, site changes stay local, multi-source is incremental;
negative — an interface with one implementation looks like
over-abstraction until the second adapter lands; cost — fixture
discipline; long-term — JSON-API sources fit the same contract
(ADR-016 proves it).

Alternatives considered: inline per-site code (why not: redesigns
touch the worker); selector configs (why not: too weak for
availability logic and validation, and configs become code anyway);
headless browser (why not: heavy, slow, worse ToS posture for the
same data).

Related ADRs: ADR-013, ADR-015, ADR-016.

## ADR-005: asynchronous checks

Status: Accepted.

Context: fetching third-party pages is slow and failure-prone (5-10
second timeouts are normal). Alternatives: synchronous scrape inside
the API request, synchronous with a long client timeout. Factors:
request latency budget, worker isolation, UX.

Decision: manual and scheduled checks create jobs and return 202; the
worker executes them from the Redis queue. The UI polls job status.

Consequences: positive — API stays fast, failures are isolated and
retryable, scheduling is uniform; negative — the UI needs job-status
handling and polling; cost — queue + reconciler + idempotency keys;
long-term — unchanged at any scale.

Alternatives considered: synchronous scrape (why not: 10-second
requests, cascading timeouts, no retry story).

Related ADRs: ADR-003, ADR-006.

## ADR-006: immutable observations

Status: Accepted.

Context: history must represent what the system observed at a specific
time, not the current best guess. Alternatives: upsert-latest rows,
mutable corrections in place. Factors: reproducibility, auditability,
simplicity of analytics.

Decision: `price_observations` is append-only. Analytics derive from
rows. Corrections require a new observation or an explicit correction
record, never an UPDATE of history.

Consequences: positive — metrics are reproducible, bugs are debuggable
from raw rows; negative — storage grows monotonically (fine at
personal scale; retention policy later); cost — none; long-term —
time-partitioning if it ever matters.

Alternatives considered: mutable latest-row (why not: destroys the
evidence the whole product is built on).

Related ADRs: ADR-002, ADR-007.

## ADR-007: integer minor units plus explicit currency exponent

Status: Accepted (revised in docs v0.1.0; prior draft example
"Rp 1.299.000 → 1299000 IDR minor units" was ambiguous because IDR
has no minor unit).

Context: money must be stored exactly, across currencies with
different decimal conventions (IDR and JPY have none, USD has two).
Alternatives: float64, NUMERIC, minor units without exponent, major
units as integers. Factors: exactness, multi-currency analytics,
formatter simplicity, frontend display.

Decision: `price_minor BIGINT` + `currency CHAR(3)` +
`currency_exponent SMALLINT NOT NULL DEFAULT 2 CHECK (0..4)`.
IDR/JPY persist exponent 0, USD exponent 2. The exponent is persisted
per observation, never re-derived from the adapter. The adapter emits
all three; the API returns all three flat; the frontend formatter
divides for display only and never feeds results back into state.

Consequences: positive — exact across currencies, self-describing
rows, one formatter everywhere; negative — every money touchpoint
must thread three fields, and alert rules must skip (not reset) on
currency mismatch; cost — a CHECK constraint and formatter unit
tests; long-term — new currencies need no schema change.

Alternatives considered: float64 (why not: inexact, disqualifying);
NUMERIC (why not: heavier, formatting still needs the exponent, no
real gain at this scale); minor units without exponent (why not:
cannot distinguish 1299000 rupiah from 12990.00 dollars).

Related ADRs: ADR-002, ADR-004, ADR-006.

## ADR-008: OpenAPI 3.1.1

Status: Accepted.

Context: the HTTP interface needs a language-agnostic contract shared
by the Go server, the React client, and contract tests. Alternatives:
3.0.x, ad-hoc documentation, code-first generation only. Factors:
JSON Schema alignment, tooling, currentness.

Decision: OpenAPI 3.1.1, with `api/openapi.yaml` in the code repo
validated against real responses in CI where practical.

Consequences: positive — one contract for server, client, and tests;
negative — the file must be kept in sync (CI check, not discipline);
cost — small; long-term — upgrade-proof while HTTP stays REST.

Alternatives considered: 3.0.x (why not: older JSON Schema dialect
for no benefit); docs-only (why not: drift is guaranteed).

Related ADRs: ADR-009.

## ADR-009: RFC 9457 error format

Status: Accepted.

Context: API errors must be machine-readable with stable types for
client handling (form errors, unsupported sources, rate limits).
Alternatives: ad-hoc `{error: string}` envelopes, RFC 7807 (older).
Factors: standardization, frontend mapping needs, obsolescence.

Decision: `application/problem+json` per RFC 9457 (which obsoletes
RFC 7807), with stable `type` URIs the frontend can switch on.

Consequences: positive — field errors map to forms, categories map to
UI states; negative — slightly more verbose than string errors;
cost — a small problem-construction helper; long-term — standard,
not ours to maintain.

Alternatives considered: string envelopes (why not: clients cannot
branch on prose); RFC 7807 (why not: superseded).

Related ADRs: ADR-008.

## ADR-010: Telegram as first notification channel

Status: Accepted.

Context: the product needs one notification integration for personal
use. Alternatives: email (SMTP), webhooks, push services. Factors:
setup cost for one user, adapter size, reliability.

Decision: Telegram Bot API first, behind a `Notifier` interface. Email
or webhooks can be added later without changing alert evaluation.

Consequences: positive — tiny integration, instant delivery to a
phone; negative — ties MVP to one provider's availability;
cost — bot token in secrets; long-term — interface makes the second
channel cheap.

Alternatives considered: email (why not: SMTP setup and spam-folder
risk for one user); webhooks (why not: needs a receiving end the user
does not have yet).

Related ADRs: none.

## ADR-011: opaque tokens — bearer access in memory, rotating refresh in cookie

Status: Accepted.

Context: the API needs authentication with login, refresh, logout,
and logout-all-devices. Alternatives: JWT access tokens, session
cookies for everything, API keys. Factors: revocation, CSRF surface,
XSS exposure, complexity. Redis is already a hard dependency
(ADR-003), so stateful tokens cost nothing extra.

Decision: opaque random access token in Redis (15-minute TTL), held in
frontend memory and sent as `Authorization: Bearer`. Opaque rotating
refresh token in Redis (30-day TTL), delivered in an httpOnly cookie
scoped to `Path=/api/v1/auth` with `SameSite=Strict` and `Secure`.
Refresh rotates (old token invalidated); reuse of a refresh token
revokes the whole token family (theft signal). All state-changing
endpoints require the bearer token, so no CSRF token is needed: the
only cookie-authenticated endpoint is `/auth/refresh`, whose response
a cross-origin attacker cannot read, and `SameSite=Strict` withholds
the cookie cross-site entirely.

Consequences: positive — instant revocation, no signing keys to
rotate, no CSRF machinery, access token invisible to XSS (memory
only); negative — every request costs one Redis read, refresh flow
adds one boot request; cost — Redis keyspace and a session table for
audit; long-term — scales with Redis, no key ceremony ever.

Alternatives considered: JWT (why not: signing-key management plus
no revocation before expiry — worse on both axes); full cookie
sessions (why not: CSRF protection needed on every mutating endpoint
for no benefit); API keys (why not: no expiry/rotation story, wrong
tool for interactive login).

Related ADRs: ADR-003, ADR-012.

## ADR-012: Argon2id for password hashing

Status: Accepted.

Context: one user account with a password needs a memory-hard hash.
Alternatives: bcrypt, scrypt, PBKDF2. Factors: GPU/ASIC resistance,
stdlib availability (`x/crypto`), parameter documentation.

Decision: Argon2id via `golang.org/x/crypto/argon2` with documented
time/memory/parallelism parameters, per-password random salt,
constant-time compare.

Consequences: positive — current best practice, tunable; negative —
slower than bcrypt (intended); cost — parameters in config, verified
in tests; long-term — re-hash on login if parameters ever change.

Alternatives considered: bcrypt (why not: weaker memory-hardness,
lower ceiling); PBKDF2 (why not: weakest of the three against GPUs).

Related ADRs: ADR-011.

## ADR-013: x/net/html plus goquery, never regex, for HTML parsing

Status: Accepted.

Context: adapters must extract prices from real-world malformed HTML.
Alternatives: regex scraping, a headless browser, site JSON APIs
where exposed. Factors: correctness on broken markup, maintenance,
performance, ToS posture.

Decision: `golang.org/x/net/html` with goquery selectors, JSON-LD /
embedded-JSON parsing where the page provides it, strict validation
of extracted values. Regex is banned for HTML parsing. Extraction
order: structured data first (JSON-LD, `__NEXT_DATA__`-style
payloads), selectors second, `unknown`/failure rather than guessing.

Consequences: positive — tolerant parsing, testable selectors,
explicit failure modes; negative — one more dependency, selectors
still rot (expected and observable); cost — fixture suite per
adapter; long-term — JSON-first order ages better than selectors.

Alternatives considered: regex (why not: incorrect on nested and
malformed markup, unmaintainable); headless browser (why not: heavy,
slow, worse ToS posture for identical data).

Related ADRs: ADR-004, ADR-015.

## ADR-014: two code repos plus docs repo

Status: Accepted.

Context: Go backend and React frontend need separate toolchains,
dependencies, and CI. Alternatives: monorepo with both, backend-only
with server-rendered UI. Factors: lanbox precedent, toolchain
independence, portfolio clarity.

Decision: `price-tracker` (Go: api, worker, scheduler) and
`price-tracker-web` (React + Vite + TypeScript), with this docs repo
as the design source. CORS allowlists the dev origin; OpenAPI is the
cross-repo contract.

Consequences: positive — independent versioning and CI, clear
ownership, matches the lanbox pattern; negative — three repos to
create and cross-reference, contract drift possible (mitigated by T5
contract tests); cost — three READMEs; long-term — standard shape,
easy to navigate.

Alternatives considered: monorepo (why not: mixed toolchains and CI
matrices in one repo buy nothing at this team size); no SPA (why not:
the product is a dashboard with charts, history, and forms).

Related ADRs: ADR-001, ADR-008.

## ADR-015: Blibli as the first source, with a fixture policy

Status: Accepted.

Context: the MVP needs one real marketplace demonstrating real HTML,
real IDR handling, and real failure modes. Alternatives: Tokopedia,
Shopee, a JSON API only. Factors: robots.txt posture (checked
2026-10-01), URL-shape stability, rate constraints, ToS scoping.
Blibli explicitly allows the product URL shape (`Allow: /p/*$`) while
forbidding query strings (`Disallow: /p/*?*`); Tokopedia allows by
omission only; Shopee imposes `Crawl-delay: 1`.

Decision: Blibli first (`https://www.blibli.com/p/<slug>`, bare URL,
no query string). Fixture policy: sanitized minimal HTML trimmed to
the selectors under test, no user data, no unrelated prices, stored
under the adapter's `testdata/`. Never use live websites in ordinary
unit tests. Scope note from PRD section 8 applies: robots.txt is
crawl permission, not a ToS grant; personal low-rate non-commercial
use only.

Consequences: positive — a defensible source choice with an explicit
permission line, real failure modes to demonstrate; negative — single
source until P1, selectors will rot; cost — fixture maintenance;
long-term — the registry makes the second source additive.

Alternatives considered: Tokopedia (why not: permission by omission
is weaker footing); Shopee (why not: 1 req/s crawl-delay constrains
the scheduler); JSON-only MVP (why not: proves nothing about the
hardest part of the product).

Related ADRs: ADR-004, ADR-013, ADR-016.

## ADR-016: dummyjson as the P1 second adapter to prove the contract

Status: Accepted.

Context: one adapter cannot distinguish a good boundary from a
hardcoded single site. Alternatives: second real marketplace now, no
second adapter ever. Factors: proof value vs Phase-3 cost, network
harmlessness of a plain JSON API.

Decision: P1 adds a dummyjson.com adapter (plain JSON: `price`,
`stock`, `availabilityStatus`). Its purpose is contract validation,
not coverage. P0 ships one source; the P0 contract is written so both
fit.

Consequences: positive — demonstrates ADR-004 generality with minimal
work, gives a deterministic source for demos; negative — a toy source
in the registry that must be labeled as such; cost — one small
adapter plus fixtures; long-term — can be replaced by a second real
marketplace without touching the domain.

Alternatives considered: second marketplace now (why not: doubles
Phase-3 scraping work against PRD principle "one source before many");
no second adapter (why not: the boundary stays unproven).

Related ADRs: ADR-004, ADR-015.
