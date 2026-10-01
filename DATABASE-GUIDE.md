# Price Tracker — Database Guide

PostgreSQL is the source of truth. Redis is transport, cache, and
coordination — never authority. All timestamps `TIMESTAMPTZ` in UTC.
All money `BIGINT` minor units plus explicit exponent.

## A. Database selection

PostgreSQL: relational, consistency-sensitive history with CHECK
constraints, partial indexes, and transactions across observation +
tracker + job writes (ADR-002). Redis holds queues, token buckets,
locks/leases, and session tokens with TTLs (ADR-003). SQLite was
rejected: concurrent workers plus a separate API process outgrow file
locking, and the managed-hosting path disappears.

## B. Schema design

```sql
CREATE TABLE sources (
    id UUID PRIMARY KEY,
    code TEXT NOT NULL UNIQUE,          -- 'blibli', 'dummyjson'
    name TEXT NOT NULL,
    host_pattern TEXT NOT NULL,         -- matched by adapter registry
    min_interval_seconds INTEGER NOT NULL DEFAULT 900,
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY,
    email CITEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,        -- Argon2id
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_login_at TIMESTAMPTZ
);

CREATE TABLE sessions (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    refresh_token_hash TEXT NOT NULL UNIQUE,
    expires_at TIMESTAMPTZ NOT NULL,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE trackers (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    source_id UUID NOT NULL REFERENCES sources(id),
    url TEXT NOT NULL,
    title TEXT,
    target_price_minor BIGINT,
    target_currency CHAR(3),
    target_currency_exponent SMALLINT NOT NULL DEFAULT 2,
    currency CHAR(3) NOT NULL,
    interval_seconds INTEGER NOT NULL,
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    last_checked_at TIMESTAMPTZ,
    next_check_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT trackers_interval_positive
        CHECK (interval_seconds >= 300),
    CONSTRAINT trackers_target_price_positive
        CHECK (target_price_minor IS NULL OR target_price_minor > 0),
    CONSTRAINT trackers_target_exponent_range
        CHECK (target_currency_exponent BETWEEN 0 AND 4),
    CONSTRAINT trackers_target_currency_present
        CHECK ((target_price_minor IS NULL) =
               (target_currency IS NULL))
);

CREATE TABLE price_observations (
    id UUID PRIMARY KEY,
    tracker_id UUID NOT NULL REFERENCES trackers(id) ON DELETE CASCADE,
    price_minor BIGINT,
    currency CHAR(3) NOT NULL,
    currency_exponent SMALLINT NOT NULL DEFAULT 2,
    availability TEXT NOT NULL,
    observed_at TIMESTAMPTZ NOT NULL,
    duration_ms INTEGER,
    source_product_ref TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT observations_price_nonnegative
        CHECK (price_minor IS NULL OR price_minor >= 0),
    CONSTRAINT observations_exponent_range
        CHECK (currency_exponent BETWEEN 0 AND 4),
    CONSTRAINT observations_availability_valid
        CHECK (availability IN ('in_stock', 'out_of_stock', 'unknown'))
);

CREATE TABLE check_jobs (
    id UUID PRIMARY KEY,
    tracker_id UUID NOT NULL REFERENCES trackers(id) ON DELETE CASCADE,
    idempotency_key TEXT NOT NULL UNIQUE,
    status TEXT NOT NULL,
    attempts INTEGER NOT NULL DEFAULT 0,
    scheduled_at TIMESTAMPTZ NOT NULL,
    started_at TIMESTAMPTZ,
    finished_at TIMESTAMPTZ,
    heartbeat_at TIMESTAMPTZ,
    error_code TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT check_jobs_status_valid
        CHECK (status IN ('pending', 'running', 'succeeded',
                          'failed', 'cancelled'))
);

CREATE TABLE alert_rules (
    id UUID PRIMARY KEY,
    tracker_id UUID NOT NULL REFERENCES trackers(id) ON DELETE CASCADE,
    type TEXT NOT NULL,
    threshold_minor BIGINT,
    threshold_percent NUMERIC(7,3),
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    state TEXT NOT NULL DEFAULT 'inactive',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT alert_rules_type_valid
        CHECK (type IN ('target_price', 'price_drop_percent',
                        'back_in_stock')),
    CONSTRAINT alert_rules_state_valid
        CHECK (state IN ('inactive', 'active'))
);

CREATE TABLE notification_deliveries (
    id UUID PRIMARY KEY,
    alert_rule_id UUID NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    idempotency_key TEXT NOT NULL UNIQUE,
    provider TEXT NOT NULL,
    status TEXT NOT NULL,
    attempt_count INTEGER NOT NULL DEFAULT 0,
    sent_at TIMESTAMPTZ,
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Note: `threshold_percent` is NUMERIC, not money — percentages are
ratios, and the no-float rule covers monetary values only.

## C. Data types

Money: `BIGINT` minor units + `CHAR(3)` currency + `SMALLINT`
exponent (ADR-007). IDR/JPY persist exponent 0; USD exponent 2. Never
`REAL`, `DOUBLE PRECISION`, or `MONEY`. Time: `TIMESTAMPTZ`, UTC
storage, presentation conversion only. Email: `CITEXT` (requires the
citext extension). IDs: `UUID` (`gen_random_uuid()`), never serial
integers (unpredictable, merge-safe). Status fields: `TEXT` with
CHECK enums (explicit, greppable, no enum-migration pain at this
scale).

## D. Indexing strategy

```sql
CREATE INDEX idx_observations_tracker_time
ON price_observations (tracker_id, observed_at DESC);

CREATE INDEX idx_trackers_due
ON trackers (next_check_at)
WHERE enabled = TRUE;

CREATE INDEX idx_jobs_pending
ON check_jobs (scheduled_at)
WHERE status = 'pending';

CREATE INDEX idx_jobs_stale_running
ON check_jobs (heartbeat_at)
WHERE status = 'running';

CREATE INDEX idx_deliveries_rule
ON notification_deliveries (alert_rule_id, created_at DESC);

CREATE INDEX idx_sessions_user
ON sessions (user_id)
WHERE revoked_at IS NULL;
```

Add indexes from actual access patterns (`EXPLAIN ANALYZE`), not
speculatively. The due-tracker and stale-running partial indexes are
the scheduler and reconciler hot paths — both are required, not
optional.

## E. Relationships

`users 1—n trackers 1—n price_observations`; `trackers 1—n
check_jobs`; `trackers 1—n alert_rules 1—n notification_deliveries`;
`users 1—n sessions`; `sources 1—n trackers`. All child rows cascade
on tracker/user delete (tracker data is user data). `sources` rows
are never deleted while referenced; disable via `enabled` instead.

## F. Performance

Hot queries and their indexes: due trackers
(`idx_trackers_due`), history pages (`idx_observations_tracker_time`),
reconciler sweep (`idx_jobs_stale_running`), delivery history
(`idx_deliveries_rule`). Analytics (lowest/highest/average/change)
compute from indexed range scans — no materialized aggregates in MVP;
short-lived dashboard caches in Redis are acceptable later with
explicit TTLs. Transaction pattern for a successful check (one
transaction): insert observation, update tracker
(`last_checked_at`, `next_check_at`), mark job `succeeded`. If these
are logically one state transition, they commit together or not at
all. Scheduler claiming uses per-tracker Redis leases, not row locks;
keep `SKIP LOCKED` for the reconciler sweep only.

## G. Migration strategy

Forward-only migrations, one logical change per migration, never edit
an applied migration, sequence-prefixed names
(`000001_create_sources.sql`). Test up/down behavior where tooling
supports it. Seed `sources` rows in a migration (Blibli enabled,
dummyjson disabled until P1). Schema changes ride with the code that
needs them; expanding guides here ride with the docs release.

## H. Monitoring & maintenance

Track: table sizes (observations growth), index bloat on the
time-ordered observation index, slow-query log for history endpoints,
connection counts per process, reconciler re-enqueue rate (a spike
means Redis or worker trouble). Retention: MVP keeps observations
indefinitely (small personal dataset); define a retention period
before scale, archive or aggregate old rows, never silently delete
history. Vacuum/autovacuum defaults are fine at this scale; revisit
with partitioning if a tracker ever exceeds ~1M observations.
