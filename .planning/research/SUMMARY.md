# Project Research Summary

**Project:** ForexFactory Weekly Calendar Harvester (FF Harvester)
**Domain:** Scheduled financial feed harvesting + Postgres archiving
**Researched:** 2026-07-09
**Confidence:** HIGH

## Executive Summary

The FF Harvester is a scheduled data pipeline that fetches the ForexFactory economic calendar feed four times daily (plus a Saturday 23:00 UTC sweep), parses and normalizes events, and archives them to an AWS RDS Postgres instance — with CSV/JSON exports for downstream consumers. This is a well-understood class of system: a single-endpoint, low-volume, synchronous ETL script driven by GitHub Actions cron. Expert practice for this pattern emphasizes raw-first archiving (persist the full payload before any parsing), deterministic upsert keys (idempotency over retries and overlapping runs), and treating parse failures as warnings rather than fatal errors.

The single most important risk is losing a week permanently. The FF feed shows only the current week; if the Saturday 23:00 UTC sweep is missed for any reason (cron skip, rate-limit, DB failure), that week's forecast and previous values are gone forever with no recovery path. Every design decision — the dedicated Saturday sweep workflow, the monitoring job, the raw_snapshots safety net, the fetch_log observability table — exists to prevent or detect this specific failure.

## Key Findings

### Stack

- **Core:** `requests` 2.32.x (HTTP), `psycopg[binary]` 3.2.x / psycopg3 (Postgres — psycopg2 is maintenance-only since 2024), stdlib `hashlib`/`re`/`decimal`/`csv`/`json` (everything else)
- **Test/infra:** `pytest` 8.x + `pytest-cov`, `python-dotenv` (dev only), plain SQL migrations, GitHub Actions (`ubuntu-latest`, Python 3.12), AWS RDS Postgres 16.x on `db.t4g.micro`
- **Avoid:** psycopg2 (deprecated), SQLAlchemy (ORM overkill), pandas (35 MB cold-start for 20 lines of stdlib), asyncio/httpx (wrong paradigm for synchronous script), Alembic (one migration doesn't warrant it)

### Table Stakes Features

- Scheduled fetch 4×/day (00:00, 06:00, 12:00, 18:00 UTC) + Saturday 23:00 UTC final sweep — gaps are permanent, no recovery source exists
- Raw snapshot storage before parsing — if a parsing bug is found, replaying from raw is the only recovery path
- Idempotent upsert with `sha1(country|normalized_title|iso_year_week)` key — same fetch twice produces identical rows; week-scoped so mid-week reschedules update in place
- Rate-limit detection via HTML body sniff (not status code) — FF returns HTTP 200 with HTML on rate-limit; naive code silently corrupts data
- Numeric value parser with raw-string fallback — parse failures are warnings, never drop an event; test DST boundary and K/M/B edge cases before any live run
- `fetch_log` per attempt — distinguishes "no events" from "silent failure"; is the MVP monitoring surface
- `actual_*` columns reserved NULL in schema now — avoids painful future migration when actuals feature ships

### Architecture

- 8-component linear pipeline: Scheduler → Fetcher → Raw Snapshot Writer → Validator → Parser → Upserter → Fetch Logger, plus a separate Export Pipeline job
- Raw Snapshot Writer is a hard precondition — if it fails, the run aborts; no orphaned unprocessed data
- Fetch Logger uses a separate DB connection so upsert transaction rollbacks cannot swallow failure log entries
- Export Pipeline is fully independent (read-only, `REPEATABLE READ` transaction, Sunday 01:00 UTC to avoid overlap with Saturday sweep)

### Critical Pitfalls

1. **Missed Saturday rollover = permanent data loss** — add a Sunday 01:00 UTC monitoring job that checks `fetch_log` for the Saturday 23:00 run and triggers `workflow_dispatch` re-fetch if absent
2. **HTML 200 rate-limit response treated as success** — validate every response parses as JSON matching expected schema before processing; classify as `rate_limited` if not
3. **Local time stored instead of UTC** — parse FF timestamps with `zoneinfo.ZoneInfo('America/New_York')`, convert to UTC, store `TIMESTAMPTZ`; unit-test DST transition samples
4. **Non-idempotent upsert on retry** — list every mutable column explicitly in `ON CONFLICT DO UPDATE SET`; test by running the same payload twice
5. **GitHub Actions cron skip** — not guaranteed to fire; 15–60 minute delays during high load; Sunday monitoring job + `workflow_dispatch` re-fetch is the mitigation
6. **Event key collision from normalization changes** — freeze normalization function before first production insert; a code change creates phantom duplicates

## Implications for Roadmap

- **Phase 1 — Core Data Pipeline:** Value parser + unit tests first (pure function, highest failure risk), then schema migration, then the full pipeline in component order. Must be idempotency-proven (run same payload twice, assert identical results) before any live run.
- **Phase 2 — Scheduling + Infrastructure:** Wire into GitHub Actions with 4×/day cron + Saturday 23:00 sweep as a dedicated workflow; provision RDS with SSL enforcement, IP whitelisting, connection limits; add Sunday monitoring job for missed sweeps.
- **Phase 3 — Export Pipeline:** Independent read-only job after harvest is proven stable; `REPEATABLE READ` transaction; atomic temp-file-then-rename; optionally commit exports to repo.
- **Research flags:** Phase 2 RDS IP whitelisting durability warrants a spot-check (GitHub Actions egress IPs change without notice); FF feed field names should be validated against a live response in Phase 1 before parser assumptions are frozen.
- **Defer:** Actual results collection, HTTP API, admin UI, event_map alias table, multi-feed support — none are needed for the two-week success metric.

## Sources

- `.planning/research/STACK.md`
- `.planning/research/FEATURES.md`
- `.planning/research/ARCHITECTURE.md`
- `.planning/research/PITFALLS.md`

---
*Research completed: 2026-07-09*
*Ready for roadmap: yes*
