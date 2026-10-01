# AGENTS.md — price-tracker-docs

Docs-only repo. English is canonical. Source of requirements and design;
code lives in `price-tracker` (Go) and `price-tracker-web` (React).

## Layout

- `PRD.md` — product requirements, 12 numbered sections, IDs `FR-001..013`.
- `ARCHITECTURE.md` — layers, flows, dependency rule, config, security.
- `DESIGN-SYSTEM.md` — React web UI contract (tokens, components, API mapping).
- `FSD.md` — per-feature behavior (`FS-01..13` mapped to FR).
- `SRS.md` — abridged IEEE 830 + traceability matrix + test tiers T1..T9.
- `ADR.md` — single file, one record per decision (`ADR-001..`).
- `SECURITY-GUIDE.md`, `CODE-STYLE-GUIDE.md`, `DATABASE-GUIDE.md`,
  `API-GUIDE.md` — technical guides, lettered sections A-K.
- `IMPLEMENTATION.md` — build order, repo blueprint, file map, test gates.
- `README.md` — index + PDF + versioning. Keep links in sync with files.
- `.github/workflows/docs-to-pdf.yml` — tag-triggered PDF release.

## ID stability

- One concept per edit; keep FR/FS/ADR IDs stable (never renumber,
  only append).
- FR numbers are owned by `SRS.md`. `PRD.md` section 5 and `FSD.md`
  reference the same IDs, never a second numbering.
- N/A sections must state why, never silently dropped.
- ADR: single `ADR.md`, append records, never rewrite decided ones (mark
  Superseded instead).

## Character rule (enforced by CI)

Verified by local pre-flight build with `pandoc/latex:3.1` + xelatex in
docs v0.1.0: these codepoints have **no glyphs** in the PDF fonts and
render as invisible holes. Never use them in any `.md` file:

- Box-drawing U+2500-U+257F (`-`, `|`, corners, junctions)
- Geometric shapes U+25A0-U+25FF (includes `v` U+25BC, `^` U+25B2)
- Math U+2264, U+2265 (`<=`, `>=` as single glyphs — write `<=` / `>=`)
- Private-use U+E000-U+F8FF (includes web-search citation artifacts;
  delete them, never paste them)
- Emoji anywhere

Allowed and proven to render: arrows U+2190-U+2193 (`<-`, `->`, `^`,
`v` direction arrows) and em-dash U+2014. Diagrams otherwise use plain
ASCII (`+`, `-`, `|`, `v`, `>`).

## Validation before push

```bash
for f in PRD ARCHITECTURE DESIGN-SYSTEM FSD SRS ADR SECURITY-GUIDE \
         CODE-STYLE-GUIDE DATABASE-GUIDE API-GUIDE IMPLEMENTATION; do
  test -s "$f.md" || exit 1
done
for f in PRD ARCHITECTURE DESIGN-SYSTEM FSD SRS ADR SECURITY-GUIDE \
         CODE-STYLE-GUIDE DATABASE-GUIDE API-GUIDE IMPLEMENTATION; do
  grep -q "$f.md" README.md || exit 1
done
rg '[t]urn0search' PRD.md ARCHITECTURE.md DESIGN-SYSTEM.md FSD.md SRS.md \
  ADR.md SECURITY-GUIDE.md CODE-STYLE-GUIDE.md DATABASE-GUIDE.md \
  API-GUIDE.md IMPLEMENTATION.md README.md
# must output nothing ([t] avoids matching this gate itself)
rg -l '[\u2500-\u257f\u25a0-\u25ff\u2264\u2265\ue000-\uf8ff]' \
  PRD.md ARCHITECTURE.md DESIGN-SYSTEM.md FSD.md SRS.md ADR.md \
  SECURITY-GUIDE.md CODE-STYLE-GUIDE.md DATABASE-GUIDE.md \
  API-GUIDE.md IMPLEMENTATION.md README.md AGENTS.md
# must output nothing
```

- Release: `git tag vX.Y.Z && git push origin vX.Y.Z` → check Actions
  green → check Release has 11 readable PDFs (`test -s
  pdf/PriceTracker-*.pdf`).
- Never add Go code, binaries, or fixtures here. New guides join the PDF
  list only via workflow + README + AGENTS update in the same PR.
- Local PDF check (same image as CI):
  `docker run --rm -v "$PWD:/data" -w /data pandoc/latex:3.1 "$f.md"
  -o "pdf/PriceTracker-$f.pdf" --pdf-engine=xelatex
  -V geometry:margin=2.5cm --toc --metadata title="Price Tracker $f"`
