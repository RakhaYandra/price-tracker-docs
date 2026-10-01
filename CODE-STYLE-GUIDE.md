# Price Tracker — Code Style Guide

Boring, explicit code over premature abstraction. Follow official Go
conventions and strict TypeScript. Consistency with Pulse (same stack)
where the guides overlap.

## A. Go rules

- `gofmt` clean, `go vet` green; `oxlint`-style import grouping:
  stdlib, then external, then internal.
- Explicit `context.Context` propagation on every I/O path; worker
  cancellation via context; no `context.Background()` below `main` and
  process entrypoints.
- Useful package names, no `utils`/`helpers` catch-alls; package names
  describe what they provide (`blibli`, `ratelimit`, `problems`).
- Doc comments on every exported declaration (effective-Go style).
- Errors: sentinel values for categories (`ErrTimeout`,
  `ErrTooLarge`, `ErrRedirectRejected`, `ErrExtractionFailed`),
  wrapped with `%w`; never compare error strings. Error categories map
  1:1 to the check `error_code` enum.
- No float for money: `int64` minor units plus `int8` exponent. Any
  `float64` touching price is a review-blocking finding.
- Time: `time.Time` in UTC internally; `TIMESTAMPTZ` at the boundary;
  never format-then-parse roundtrips.
- Concurrency: bounded (`errgroup` with `SetLimit` or semaphore);
  goroutine lifetimes owned by a parent context; no fire-and-forget.
- Dependency rule (enforced by `goimports`-level review plus an import
  test): `httpapi -> app -> domain`; `app -> ports`; `infra ->
  ports`; `domain` imports stdlib only. No `infra` import from
  `domain` or `app`.
- Tests colocated (`foo_test.go`); table-driven; no live network in
  unit tests (fixtures under `testdata/`).

## B. React + Vite + TypeScript

Same toolchain as `shiftbase-web`/`lifeos-web`: React 19, Vite 8,
strict TypeScript (`noExplicitAny`), `oxlint`. React 19 rules: no
`forwardRef` (use `ComponentProps`), `data-slot` on primitives, no
legacy context API.

- Server is the source of truth. Never `fetch` directly in a
  component: use the typed client (`src/lib/api.ts`, throws
  `ApiProblem`) through TanStack Query hooks with the query-key
  factory (`src/lib/query.ts`). Invalidate by key after mutations;
  poll job status with `refetchInterval` until terminal, then
  invalidate the tracker queries.
- Forms: `react-hook-form` + `zod`. Server problems map to inline
  field errors via `fieldErrors`; network/5xx go to sonner toasts.
  Never toast a field error, never inline a global failure.
- Styling: Tailwind utilities plus `cn()` (`clsx` +
  `tailwind-merge`); no inline `style` except Recharts props fed from
  tokens. Colors only from `@theme` tokens.
- Recharts: never inside loops or `.map`; data memoized; colors from
  tokens (`var(--chart-1)`, never wrapped); `isAnimationActive`
  respects `motion-reduce`; every chart has role/aria-label plus a
  visually-hidden data table.
- Money display: only `formatMinor()` from `src/lib/format.ts`. The
  function is display-only; its output never re-enters state. Unit
  tests assert exact strings for IDR (exponent 0) and USD (exponent 2).
- Auth: access token lives in the AuthProvider (memory), never in
  storage; boot calls refresh once with a "restoring session" state;
  401 triggers one refresh-and-retry, then logout redirect.

## C. Go example

```go
// Package blibli adapts Blibli product pages to ObservationData.
// It owns all Blibli-specific selectors; nothing outside this
// package may reference them.
package blibli

// Adapter fetches and parses Blibli product pages.
type Adapter struct {
    client *http.Client // timeouts, size cap, redirect policy set by constructor
    clock  Clock
}

// Fetch returns normalized observation data or a categorized error.
// Unparsable availability yields Available=false with Availability
// "unknown"; it is never guessed.
func (a *Adapter) Fetch(ctx context.Context, target Target) (ports.ObservationData, error) {
    // 1. validate URL shape (bare /p/<slug>, no query)
    // 2. GET with ctx deadline (redirects revalidated per hop)
    // 3. parse structured data first (JSON-LD), selectors second
    // 4. validate price (positive, sane magnitude) and currency
}
```

## D. Common standards

- SQL: always list columns, parameterize everything, no string-built
  queries outside the migration files; `EXPLAIN ANALYZE` slow queries
  before adding indexes.
- API: resource nouns, plural collections, `/api/v1` prefix,
  RFC 9457 problems, cursor pagination, ISO 8601 UTC timestamps,
  OpenAPI updated in the same PR as the handler.
- Naming: `*_at` for timestamps, `*_minor` for minor-unit money,
  `*_ms` for durations, `idempotency_key` for dedup keys.
- Comments explain why, not what. No commented-out code. No TODO
  without an owner and a milestone (`TODO(M4, user): ...`).
- Frontend copy follows DESIGN-SYSTEM section 10: calm, precise,
  actionable.

## E. Tools & enforcement

Go repo: `gofmt -l .` empty, `go vet ./...`, `go test ./...`, plus the
import-direction test and the `float64`-near-money grep in CI.
Web repo: `tsc --noEmit`, `oxlint`, `vite build`, `vitest run`,
Playwright e2e on push. Both: secret scan on push, dependency audit
monthly. Docs repo (here): the gates in AGENTS.md.
