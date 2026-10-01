# Price Tracker — Security Guide

Single-user personal tool that fetches untrusted third-party HTML on a
schedule, stores credentials and sessions, and sends Telegram messages.
The threat model assumes the network is hostile, site markup is
adversarial, and the operator (the user) is trusted.

## A. Authentication & Authorization

Passwords: Argon2id with documented parameters, random salt per
password, constant-time compare. Generic login error only ("invalid
email or password") — never distinguish unknown email from wrong
password. Login rate limit: 5 attempts per minute per IP, then
exponential lockout; lockout state in Redis so restarts do not clear
it.

Sessions: opaque random access token (Redis, 15-minute TTL) in
frontend memory, sent as `Authorization: Bearer`. Opaque rotating
refresh token (Redis, 30-day TTL) in an httpOnly cookie with
`Path=/api/v1/auth`, `SameSite=Strict`, `Secure`. Refresh rotates;
reuse revokes the token family. Logout revokes both; logout-all
revokes every session row for the user.

Authorization: every `/api/v1/*` endpoint except login/refresh
requires a valid bearer token via `RequireAuth` middleware. MVP has
one account, so checks are trivially satisfied — but object-level
authorization checks must be structurally present and must not be
removed when multi-user support arrives (OWASP API1: broken
object-level authorization is the top API risk, and "single user so
skip it" is the expensive mistake). Every mutating request is
attributed to the actor (NFR-008). New endpoints must fail closed:
deny by default, allow explicitly.

## B. Data Security

Money and observations: no floating point anywhere; see ADR-007.
Timestamps UTC (`TIMESTAMPTZ`); timezone conversion at presentation
only. Page content: never stored beyond sanitized minimal fixtures;
operational tables carry metadata (status, duration, category), not
bodies. User data minimization: sessions table keeps only what audit
and revocation need (no full user agents, no raw IPs beyond coarse
rate-limit keys with short TTLs). Backups: PostgreSQL dumps contain
password hashes (Argon2id, safe) and Telegram tokens (secret — encrypt
backups or exclude and re-enter). Redis RDB contains live tokens —
restrict file permissions and never ship RDB files anywhere.

## C. API Security

Input validation on every endpoint: URL shape, enum values, numeric
ranges, string lengths, pagination bounds. Authentication, object-level
authorization, rate limits, and content-size limits per OWASP API
Security Top 10. SSRF-relevant risks (unrestricted resource
consumption, unsafe consumption of APIs) are handled in section D.
Error responses are RFC 9457 problems with stable types; they never
carry stack traces, SQL errors, upstream secrets, or internal network
details. Maintain an endpoint inventory (from OpenAPI) and review it
per release — undocumented endpoints are findings.

## D. Infrastructure Security

Scraper egress is the highest-risk surface. Mandatory controls:

1. Scheme allowlist: HTTPS only (HTTP only if explicitly configured).
2. Host must match a `host_pattern` in the `sources` table (MVP: two
   known hosts). This is the primary control — an allowlist of known
   hosts is strictly stronger than validating arbitrary user URLs.
3. Resolve DNS, then reject: RFC1918 (`10/8`, `172.16/12`,
   `192.168/16`), loopback (`127/8`, `::1`), link-local
   (`169.254/16`, `fe80::/10`), CGNAT (`100.64/10`), multicast, and
   `0.0.0.0`. Re-resolve and re-check on every redirect hop (DNS
   rebinding defense).
4. Custom `CheckRedirect`: maximum 3 hops, re-validate scheme/host/IP
   per hop, scheme must stay HTTPS (no https-to-http downgrade).
5. Response cap 2 MiB (`http.MaxBytesReader` semantics on the client
   side); total deadline 10 s, 5 s per hop.
6. Identifiable User-Agent with contact; per-host Redis token bucket;
   honor `Crawl-delay` where published.
7. Never trust third-party HTML: parse with goquery, extract via
   structured data first, validate magnitudes, fail to
   `extraction_failed` rather than guess.

Filesystem: the scraper writes nothing to disk except logs. Uploads do
not exist in MVP, so there is no file-upload attack surface.

## E. Secrets Management

Secrets: Telegram bot token, session signing none (opaque tokens),
Redis password, PostgreSQL password. Local: `.env` file, never
committed (`.env.example` documents names only). Production:
environment or secret manager injection. Rotation: Telegram token
rotatable via settings with a test action; Redis/Postgres passwords
rotatable with a restart. Logs never contain secrets; config dumps
redact values. CI scans for committed secrets (`gitleaks` or
equivalent) on every push.

## F. Incident Response

Adapter compromise (site serves malicious markup): the parser is a
pure function over bytes — worst case is a failed check, recorded and
alertable. Response: disable the source row, inspect fixtures, fix
selectors, re-enable. Credential compromise: rotate password
(re-hash), revoke all sessions, rotate Telegram token, check audit
trail for unfamiliar mutations. Redis flush: reconciler re-enqueues
from PostgreSQL; verify no duplicate notifications via idempotency
keys, then investigate the flush cause. Upstream ToS change: pause all
trackers for that source the same day; docs record the decision.

## G. Security Testing

Mandatory suite (T8): scheme bypass (`http://`), cloud metadata
(`169.254.169.254`), loopback (`127.0.0.1`, `[::1]`, `0.0.0.0`),
redirect-to-internal (302 to private IP), redirect-chain overflow (4+
hops), DNS rebinding to private IP, https-to-http downgrade, 3 MiB
body rejection, oversized-URL rejection, non-matching-host rejection.
Auth: unauthenticated sweep of every endpoint (all 401), wrong-
password indistinguishability, lockout on 6th rapid attempt, refresh
reuse revokes family, Bearer tampering rejected. Rate limits:
per-host bucket and login bucket under load. Secret scan and
dependency scan in CI.

## H. Compliance

No payment data, no third-party personal data stored, no tracking.
robots.txt is crawl permission, not a ToS grant; marketplace terms
generally forbid automated scraping, so the project scope is personal,
low-rate, non-commercial use with no CAPTCHA bypass, no auth bypass,
and no proxy rotation (PRD section 8). If a source's terms change,
section F applies. Telegram messages contain only the user's own
tracker data.
