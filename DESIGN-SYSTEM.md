# Price Tracker — Design System

Data-first, calm, utilitarian interface. Price changes must be
immediately understandable. No decorative dashboards. Every state has an
explicit visual representation. Desktop is the primary dashboard
target; mobile stays usable. Stack: Tailwind CSS v4 + shadcn
(Radix primitives, `new-york` style) + lucide-react + Recharts for the
price chart.

## 1. Color Palette

Colors are specified in OKLCH (shadcn v4 migrated HSL to OKLCH; the
docs follow what `shadcn init` generates). MVP is light-only; the
`.dark` block is reserved in `styles.css` and documented here so the
structure survives a later dark mode.

Semantic tokens (names are normative; values are set once in
`@theme inline`):

```text
--background / --foreground
--card / --card-foreground
--muted / --muted-foreground
--border / --input / --ring
--primary / --primary-foreground
--destructive / --destructive-foreground
--success / --warning / --info          (status only, see section 5)
--chart-1 / --chart-2 / --chart-grid / --chart-axis
```

Rules: never hard-code colors per component; always use a semantic
token. Semantic status colors are used only for status, never for
decoration. Exact values live centrally in CSS variables.

## 2. Typography

```text
font.family.sans = "Inter Variable", system-ui, sans-serif
font.family.mono = "JetBrains Mono Variable", ui-monospace, monospace
font.size.xs   = 12px
font.size.sm   = 14px
font.size.md   = 16px
font.size.lg   = 20px
font.size.xl   = 28px
font.size.2xl  = 36px
```

Fonts are self-hosted via `@fontsource-variable` packages. Monospace
is used for prices in tables, technical values (durations, IDs), and
code. Prices on cards use the sans family at `lg` or `xl` for
scannability.

## 3. Spacing / Grid

Base unit 4px, matching the Tailwind scale exactly:

```text
xs = 4   sm = 8   md = 12   lg = 16   xl = 24   2xl = 32   3xl = 48
```

Radius: `sm = 6px`, `md = 10px`, `lg = 14px`, `pill = 999px`, set via
`--radius-*` in `@theme`. Layout: max content width 1200px, 12-column
grid on desktop, sidebar 240px fixed.

## 4. Icons

lucide-react only. One icon per status concept, used consistently:

```text
In stock        CircleCheck
Out of stock    CircleX
Unknown         CircleHelp
Check running   Loader (spin, motion-reduce: static)
Check failed    TriangleAlert
Price down      TrendingDown
Price up        TrendingUp
No change       Minus
Target hit      BellRing
Paused          Pause
Settings        Settings
```

Icons always pair with a text label; status must never rely on icon or
color alone (see section 7).

## 5. Components

shadcn primitives are copied source owned by the web repo
(`src/components/ui/`), React 19 style: no `forwardRef`, `data-slot`
attributes, strict TypeScript. Toast uses `sonner`, not the deprecated
Radix Toast.

Primitives: Button, Input, Label, Select, Dialog, AlertDialog,
Tabs, Table, Tooltip, DropdownMenu, Popover, Skeleton, Badge, Card,
Form.

Price Tracker composites (`src/components/`):

- AppShell, Sidebar, TopBar
- StatCard (current / lowest / highest / average)
- ProductTable (monospace prices, availability badge, check freshness)
- ProductForm (URL + target price + interval; field errors from
  RFC 9457 problem details)
- AlertRuleForm (type + threshold; one rule per form)
- PriceChart (Recharts; see section 11)
- CheckStatus (pending / running / retrying / succeeded / failed)
- PriceBadge (current vs previous with arrow + percentage)
- AvailabilityBadge (in_stock / out_of_stock / unknown)
- FilterBar (status + source + search)
- EmptyState, ErrorState (with retry action), LoadingSkeleton
- ConfirmDialog (destructive actions only), Toast via sonner

Status rules:

```text
Availability:  In stock -> success, Out of stock -> warning, Unknown -> neutral
Check:         Success -> success, Running -> info, Retrying -> warning,
               Failed -> danger, Never checked -> neutral
Price:         "v 8.2% vs previous" / "^ 4.1% vs previous" / "No change"
```

Do not imply that lower is always good; use descriptive labels only.

Login page: email + password, single generic error for bad credentials
(never "email not found"), visible rate-limit state ("too many
attempts, try again in N minutes"), caps-lock hint. Boot state
"restoring session" while the first refresh runs.

## 6. Patterns

Query state (TanStack Query) maps to UI states from section 9:

```text
isPending                -> loading (skeleton, never blank page)
error                    -> error (message + retry button)
data empty               -> empty (illustration + primary action)
data + isFetching        -> success + subtle refreshing indicator
mutation pending         -> optimistic update + disabled controls
401                      -> redirect to login (session expired)
problem+json fieldErrors -> inline form errors, no toast
```

Form errors come from the server as RFC 9457 problems; the API client
maps `fieldErrors` to form fields. Global failures (network, 5xx) go to
sonner toasts, never to form fields. Delete and pause are one-click
with undo where cheap, ConfirmDialog where destructive and
irreversible (delete).

## 7. Accessibility Guidelines

- Keyboard navigable; visible focus ring (`--ring`) on all interactive
  elements.
- Form labels for every input; accessible names on icon-only buttons.
- Contrast meets WCAG AA on text; status never relies on color alone
  (icon + text always present).
- Semantic HTML: `table` for data, `nav` for sidebar, `main` landmark.
- PriceChart has role="img" with an aria-label plus a visually-hidden
  data table as its textual summary; the same summary is available as
  the observations table below the chart.
- `prefers-reduced-motion`: disable spin/pulse animations and chart
  animation (Recharts `isAnimationActive={false}`).
- Live region for check-status transitions and toast announcements.

## 8. Animation / Motion

Tailwind transitions plus `tw-animate-css` (shadcn v4 default; the old
`tailwindcss-animate` plugin is deprecated). Rules: 150-250 ms for
micro-interactions, no layout-shift animation, no decorative motion.
Chart animation is on for sighted users, off under `motion-reduce`.
Loading skeletons pulse; spinners rotate; nothing else moves on its
own.

## 9. Responsive Breakpoints

Tailwind defaults, mapped to three layouts:

```text
Desktop (>= 1024px): sidebar + content, table view
Tablet  (640-1023):  collapsible sidebar, responsive table
Mobile  (< 640px):   stacked cards, horizontal chart scroll if needed,
                     bottom or top navigation
```

Breakpoints: `sm 640`, `md 768`, `lg 1024`, `xl 1280`. The Tracker
Detail page keeps "current price / alert state / movement / freshness"
order at every width.

## 10. Brand Voice & Tone

Calm, utilitarian, precise. Errors say what happened and what to do
next ("Check failed: source returned 503. Will retry in 15 minutes.").
Never blame the user. Numbers are exact (prices with currency,
durations in ms/s). Empty states suggest the single next action.
Notification copy is one factual sentence plus the price.

## 11. Contract with code

Tailwind v4 is CSS-first: tokens live in `src/styles.css` under
`@theme inline`, mapping each section-1 token to a CSS variable. No
`tailwind.config.js`. Example shape (values set once):

```css
@import "tailwindcss";
@import "tw-animate-css";

:root {
  --background: oklch(...);
  --foreground: oklch(...);
  --chart-1: oklch(...);
}
@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-chart-1: var(--chart-1);
}
```

Chart color rule: Recharts props reference `var(--chart-1)` directly.
Never wrap in `hsl()` (v3 and earlier shadcn pattern does not apply to
v4/OKLCH). Recharts components are never rendered inside loops or
`.map`; chart data is memoized; colors come only from tokens.

Money formatting is the keystone display function
(`src/lib/format.ts`, unit-tested):

```text
formatMinor(1299000, 'IDR', 0) === 'Rp 1.299.000'
formatMinor(129900,  'USD', 2) === '$1,299.00'
```

Rule: the formatter divides `minor / 10**exponent` for display only
and the result is never fed back into state. State stays integer.
This keeps any float drift display-only by construction.
