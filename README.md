# Price Tracker Docs

[![docs](https://github.com/RakhaYandra/price-tracker-docs/actions/workflows/docs-to-pdf.yml/badge.svg)](https://github.com/RakhaYandra/price-tracker-docs/releases)

Official documentation for **Price Tracker**, a personal price monitoring
system (English). This repo is the source of requirements and design; code
lives in `price-tracker` (Go API, scheduler, workers) and
`price-tracker-web` (React UI).

| Document | Content |
|---|---|
| [PRD.md](PRD.md) | Product Requirements — 12 sections, FR-001..013 ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture, layers, flows ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [DESIGN-SYSTEM.md](DESIGN-SYSTEM.md) | React web UI contract — tokens, components ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [FSD.md](FSD.md) | Functional Specification — per feature, FS-01..13 mapped to FR ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [SRS.md](SRS.md) | Abridged IEEE 830 — FR traceability matrix, test tiers T1..T9 ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [ADR.md](ADR.md) | Architecture Decision Records — 16 decisions ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [SECURITY-GUIDE.md](SECURITY-GUIDE.md) | Security model, SSRF defense, auth ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [CODE-STYLE-GUIDE.md](CODE-STYLE-GUIDE.md) | Go + React + TypeScript conventions ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [DATABASE-GUIDE.md](DATABASE-GUIDE.md) | PostgreSQL schema, money model, migrations ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [API-GUIDE.md](API-GUIDE.md) | Endpoint reference + conventions ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [IMPLEMENTATION.md](IMPLEMENTATION.md) | Build order, file map, formats, test gates ([PDF](https://github.com/RakhaYandra/price-tracker-docs/releases/latest)) |
| [AGENTS.md](AGENTS.md) | Agent instructions for this repo (markdown only) |

## PDF

Each `v*` tag triggers the `docs-to-pdf` workflow → all 11 guides upload as
**release assets**. Download them on the
[Releases](../../releases) page. AGENTS and README stay markdown-only.

Character rule: docs are ASCII-safe except arrows (U+2190-U+2193) and
em-dash, verified by local pre-flight build with `pandoc/latex:3.1`.
See AGENTS.md. Box-drawing, geometric shapes, math glyphs, private-use
characters, and emoji are forbidden (they have no PDF glyphs).

## Versioning

| Version | Date | Content |
|---|---|---|
| v0.1.0 | 2026-10-01 | Initial release: PRD, ARCHITECTURE, DESIGN-SYSTEM, FSD, SRS, ADR (16), SECURITY/CODE/DATABASE/API guides, IMPLEMENTATION + PDFs (11 total) |

## Draft changes (recorded in v0.1.0)

Decisions that changed relative to the pre-release engineering drafts:

1. Redis moved from optional infrastructure to a hard MVP dependency
   (job queue + rate limiting + session store + locks). See ADR-003.
2. Money model clarified: integer minor units plus an explicit
   `currency_exponent` column (0 for IDR/JPY, 2 for USD). The draft
   example was ambiguous. See ADR-007.
3. Full login added to scope. The drafts assumed a single user without
   auth; the release requires opaque access tokens (memory) plus
   rotating refresh tokens (httpOnly cookie). See ADR-011, FR-012,
   NFR-008 (now mandatory), NFR-011.
4. First source named: Blibli. Second adapter (dummyjson JSON API) scoped
   as P1 contract validation. See ADR-015, ADR-016.

## Roadmap (docs)

Done: PRD, ARCHITECTURE, DESIGN-SYSTEM, PDF pipeline, AGENTS, FSD, SRS,
ADR, SECURITY-GUIDE, CODE-STYLE-GUIDE, DATABASE-GUIDE, API-GUIDE,
IMPLEMENTATION. Next: code repos `price-tracker` and `price-tracker-web`
implementation.
