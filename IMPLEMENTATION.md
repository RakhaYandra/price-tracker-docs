# Price Tracker — Implementation Blueprint

Build order, repo blueprint, file map, config, and test gates for the
code repos: `price-tracker` (Go) and `price-tracker-web` (React +
Vite + TypeScript). Normative behavior lives in FSD/API-GUIDE; this
file says what to build first and how to prove it.

## 1. Repo blueprint

```text
price-tracker/
+-- cmd/api/main.go                 # thin: wire config, start server
+-- cmd/worker/main.go              # thin: consume pt:q:check
+-- cmd/scheduler/main.go           # thin: tick, claim, enqueue, reconcile
+-- cmd/notifier/main.go            # thin: consume pt:q:notify (may merge into worker in M0)
+-- internal/domain/
+--   product/      # tracker entity + validation (FR-001, FR-002, FR-013)
+--   observation/  # ObservationData, money value object (FR-005..007)
+--   alert/        # rule evaluation + dedup state machine (FR-008, FR-009)
+--   session/      # login/refresh/logout logic (FR-012)
+-- internal/app/                   # use cases: CreateTracker, TriggerCheck,
+--                                 # EvaluateRules, Authenticate, ... (no HTTP, no SQL)
+-- internal/ports/                 # SourceAdapter, Repos, Notifier, Queue,
+--                                 # RateLimiter, Clock, TokenStore (interfaces)
+-- internal/infra/
+--   postgres/     # repos + migrations/ (T2)
+--   redis/        # queue, ratelimit, lock, session (T4)
+--   httpclient/   # SSRF-hardened client: allowlist, per-hop checks, caps (T8)
+--   adapters/blibli/    # Match + Fetch + testdata/ fixtures (T3)
+--   adapters/dummyjson/ # P1 (ADR-016)
+--   telegram/     # Notifier implementation (T4)
+--   auth/         # Argon2id + opaque token minting (T4, T8)
+--   clock/        # real + fake clock (tests)
+-- internal/httpapi/               # router, handlers, middleware, problems (T4, T5)
+-- migrations/000001_*.sql
+-- api/openapi.yaml                # contract, validated in CI (T5)
+-- Makefile  go.mod  README.md  .gitignore
```

Import rule: `httpapi -> app -> domain`; `app -> ports`;
`infra -> ports`; `domain` imports stdlib only. Enforced by an import
test, not by convention alone. Layout: cmd thin, everything in
`internal`, no `pkg/` (nothing reusable externally), tests colocated.

```text
price-tracker-web/                  # React 19 + Vite 8 + TS strict
+-- src/main.tsx                    # QueryClientProvider > AuthProvider > Router
+-- src/App.tsx                     # route table, 8 routes
+-- src/styles.css                  # tailwind import + @theme inline tokens
+-- src/components.json            # shadcn config
+-- src/components/ui/              # copied shadcn source, owned here
+-- src/components/                 # AppShell, Sidebar, tables, forms, PriceChart, ...
+-- src/features/{auth,dashboard,trackers,observations,alerts,operations,settings}/
+-- src/lib/{api.ts,format.ts,query.ts,cn.ts}
+-- e2e/{auth,tracker,alert,history}.spec.ts   # @playwright/test (T7)
+-- index.html  package.json  vite.config.ts  tsconfig.json  .env.example
+-- .oxlintrc.json
+-- dist/                           # build output (gitignored)
```

FE grows to shared/ only if a second cross-feature composite appears;
colocation first. Strict TypeScript, no explicit `any`. e2e stays in
the web repo and boots a real stack.

## 2. Config format

Environment variables, precedence flags > env > file > defaults:

```text
DATABASE_URL=postgres://...
REDIS_URL=redis://...
TELEGRAM_BOT_TOKEN=...            # secret, never logged
TELEGRAM_CHAT_ID=...
SESSION_ACCESS_TTL=15m
SESSION_REFRESH_TTL=720h
WORKER_CONCURRENCY=4
SCHEDULER_TICK=30s
SCRAPER_TIMEOUT=10s
SCRAPER_MAX_BYTES=2097152
MIN_INTERVAL_SECONDS=300
WEB_ORIGIN=http://localhost:5173  # CORS allowlist, exact match
```

`.env.example` documents names only, no values.

## 3. File map

Backend: every domain package maps to FS features (see FSD); every
infra package maps to a port; every handler maps to an API-GUIDE
endpoint. Frontend: every `features/*` directory maps to FS-01/12/13
pages; every `components/*` composite maps to a DESIGN-SYSTEM section
5 entry; `lib/format.ts` implements the money contract (T6); `lib/
api.ts` implements the problem-mapping contract (T5/T6).

## 4. Build order

```text
M0 Repository + contracts
M1 Database + domain + auth skeleton
M2 Auth + Blibli adapter + manual check
M3 Scheduler + queue + observation pipeline
M4 Alerts + Telegram + dashboard
M5 Operations + settings + hardening
M6 P1 second adapter + polish
```

Do not start with dashboard polish. Do not add the second adapter
before the first is stable. Do not add notification channels before
Telegram delivers exactly-once per transition.

## 5. Tasks per milestone

M0 repo + contracts (DoD: `make test/lint/build` green, Compose
starts the stack): 1. Go module + cmd skeletons + Compose
(postgres, redis, api, worker, scheduler, web). 2. `api/openapi.yaml`
skeleton with auth + tracker + check endpoints. 3. `Makefile`
(test/lint/vet/build) + CI (fmt, vet, test, secret scan). 4. ports
interfaces (SourceAdapter, Repos, Notifier, Queue). 5. web scaffold:
Vite + Tailwind + shadcn init, router, AuthProvider shell. 6. README
setup + env docs in both code repos.

M1 database + domain (DoD: migrations apply cleanly, pure domain
tests green): 1. migrations 000001-00000n (all tables + indexes +
source seeds). 2. repository implementations + T2 tests
(testcontainers). 3. domain packages (product validation,
ObservationData + money value, alert state machine, session logic) +
T1 tests. 4. clock fake + idempotency-key helper.

M2 auth + adapter + manual check (DoD: login roundtrip works, one
fixture-passing adapter, manual check returns 202 then exactly one
observation): 1. Argon2id + opaque tokens + RequireAuth + login/
refresh/logout/me. 2. login rate limit + lockout. 3. SSRF-hardened
httpclient + redirect policy + caps + T8 matrix. 4. Blibli adapter +
fixtures (price, out-of-stock, drift) + T3, no live network. 5.
`POST /trackers/{id}/checks` -> queue -> worker -> observation
transaction. 6. LoginPage + boot-refresh + 401-retry + e2e auth.

M3 scheduler + pipeline (DoD: due tracker checked within one
interval, no duplicate jobs, Redis flush loses nothing): 1.
scheduler tick + leases + leader lock + enqueue. 2. worker consume +
bounded concurrency + backoff/jitter + DLQ. 3. reconciler sweep +
stale-heartbeat re-enqueue. 4. alert evaluation + dedup
(FALSE->TRUE fires once, currency mismatch skips). 5. trackers CRUD +
pause/resume + T5 contract tests.

M4 alerts + telegram + dashboard (DoD: exactly one Telegram message
on first TRUE, dedup proven, dashboard loads with skeletons): 1.
Telegram notifier + idempotency + retry/backoff + delivery status.
2. dashboard queries + StatCard + attention lists. 3. tracker detail:
PriceChart (Recharts 3.x, token colors) + observations table + a11y
summary. 4. forms with fieldErrors mapping (react-hook-form + zod).
5. `formatMinor` + T6 unit tests (IDR/USD exact strings).

M5 operations + settings + hardening (DoD: SSRF matrix green,
security tests green, settings roundtrip works): 1. operations
endpoints + failures view. 2. settings + test-notification action.
3. full T8 suite (SSRF, authz sweep, rate limits, secret scan).
4. reliability drills: worker restart, DB restart, Redis flush,
source 5xx, malformed HTML, notification failure. 5. docs: README
complete in both code repos.

M6 P1 (DoD: two adapters, contract generality demonstrated): 1.
dummyjson adapter + fixtures. 2. lowest-in-period + back-in-stock
rules. 3. richer diagnostics per source. 4. T9 benchmarks + k6 smoke.

## 6. Test plan

- T1: `go test ./internal/domain/...` (table-driven, no I/O).
- T2: repository tests with testcontainers PostgreSQL, incl. partial
  indexes and the check transaction.
- T3: adapter fixture tests, no network (`go test
  ./internal/infra/adapters/...` with `-short` safe offline).
- T4: API + PostgreSQL + Redis integration (Compose or
  testcontainers), incl. auth roundtrip, queue consume, reconciler.
- T5: OpenAPI response validation in CI (real responses vs
  `api/openapi.yaml`).
- T6: `vitest run` (formatMinor exact strings, query keys, cn).
- T7: Playwright e2e in `price-tracker-web/e2e/` (auth, tracker,
  alert, history) against a real stack; CI on push.
- T8: SSRF matrix + auth sweep + rate limits + secret scan.
- T9: benchmarks (API list, history query, adapter parse,
  concurrent workers) + k6 smoke.
- Gates: `gofmt -l .` empty, `go vet ./...` green, full `go test
  ./...` green; web: `tsc --noEmit`, `oxlint`, `vite build`,
  `vitest run` green.

## 7. Makefile / CI (code repos)

```make
build: go build -o bin/api ./cmd/api && go build -o bin/worker ./cmd/worker && go build -o bin/scheduler ./cmd/scheduler
test:  go test ./...
vet:   go vet ./...
fmt:   gofmt -l .
```

CI (Go repo, separate from docs-PDF CI): fmt + vet + test + import-
direction test + float-near-money grep + SSRF matrix + secret scan on
every push. Web CI: `tsc --noEmit` + `oxlint` + `vite build` +
`vitest run` + Playwright e2e on push.
