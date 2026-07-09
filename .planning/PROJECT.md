# FF Harvester

## What This Is

A scheduled Python script that downloads the ForexFactory weekly economic calendar feed four times a day and permanently archives every major-currency event — title, scheduled time, impact, forecast, and previous value — into a Postgres database. The tool exists because the feed only ever shows the current week: once a week rolls over, the forecast and previous values it published are gone forever. The primary output is a steadily growing `events` table plus CSV/JSON exports that downstream consumers can download and query.

## Core Value

Never miss a week, never corrupt a value — the archive is only useful if it's complete and trustworthy.

## Business Context

- **Customer**: Downstream consumers who need a historical ForexFactory dataset (forecasts + previous values) that would otherwise be unrecoverable once weeks roll over
- **Revenue model**: Internal data product / open dataset — no direct monetization in MVP
- **Success metric**: Two consecutive unattended weeks with zero missed Saturday sweeps, zero duplicates, and clean exports available

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Fetch `https://nfs.faireconomy.media/ff_calendar_thisweek.json` on a 4×/day cron schedule (00:00, 06:00, 12:00, 18:00 UTC)
- [ ] Run a Saturday final sweep at 23:00 UTC to capture the closing week's last state before rollover
- [ ] Filter to only the 8 major currency codes: USD, EUR, GBP, JPY, AUD, NZD, CAD, CHF
- [ ] Convert all timestamps to UTC on ingest; never store local time
- [ ] Save every successful download as a raw snapshot (full JSON payload + metadata) before any processing
- [ ] Parse forecast/previous display strings into a numeric value + unit, storing both raw and parsed forms
- [ ] Upsert events using a deterministic key: sha1(country | normalized_title | ISO year-week), updating in place if an event is rescheduled within the same week
- [ ] Write a `fetch_log` row for every download attempt (outcome, event counts, HTTP status)
- [ ] Handle rate-limiting gracefully: detect HTML response, retry once after 10 minutes, give up and await next schedule
- [ ] Export the full `events` table to `events.csv` and `events.json` on demand and on a weekly Sunday schedule
- [ ] Provision AWS RDS Postgres (starting fresh) with SSL-only access restricted to GitHub Actions egress
- [ ] Apply database migration creating `raw_snapshots`, `events`, and `fetch_log` tables with `actual_*` columns reserved for future actuals feature
- [ ] Unit-test the value parser against real feed samples (including blank, `<0.1%`, K/M/B suffixes) before any other code

### Out of Scope

- Collecting actual results — `actual_*` columns exist in schema but stay NULL for MVP
- A proper HTTP API — exports are file-based (CSV/JSON) for MVP
- Real-time or sub-hourly harvesting — 4×/day is sufficient
- OAuth or admin UI — no web interface of any kind
- Next-week probe — listed as optional; deferred unless early testing confirms the endpoint exists
- `event_map` alias table (for title rewording across weeks) — deferred to a later iteration

## Context

- **Feed behavior**: Only shows the current week; rolls over around the weekend. Approximately 2 requests per 5 minutes per IP before rate limiting kicks in (returns HTML instead of JSON). URL/format could change without notice — strict validation catches this early.
- **Data model decision**: Events are identified by `event_key = sha1(country | normalized_title | ISO year-week)` — week-scoped, not timestamp-scoped, so mid-week reschedules update the existing row rather than create a duplicate.
- **Raw snapshots as safety net**: All raw payloads are saved before parsing. If a parsing bug is discovered, every event can be re-processed from these snapshots — no data is lost permanently.
- **Downstream consumers**: Others will query the DB or consume the CSV/JSON exports as a data product. Reliability and correctness of the archive matter more than throughput.
- **Future actuals**: FMP (Financial Modeling Prep) with FRED as override is planned for a later phase. The `actual_*` columns and the schema design are pre-built to make that a column-fill operation, not a rebuild.

## Constraints

- **Tech stack**: Python + GitHub Actions + AWS RDS Postgres — no servers, no framework, minimal deps (requests, psycopg)
- **Rate limit**: Minimum 5 minutes between any two requests to the FF feed; max 1 retry per run
- **AWS**: Starting from scratch — RDS provisioning is part of this project; use `db.t4g.micro` (free tier eligible for 12 months)
- **DB access**: RDS must not be open to the public internet; connections restricted to GitHub Actions egress IPs, SSL required, credentials in GitHub Actions secrets
- **Parsing**: Parse failures are warnings only — they never drop an event; raw string is always stored regardless

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Week-scoped event key (ISO year-week, not exact timestamp) | Prevents duplicate rows when events are rescheduled mid-week | — Pending |
| Store raw snapshots unconditionally | Enables re-parsing from scratch if bugs are found; storage cost is negligible | — Pending |
| Reserve `actual_*` columns now | Future actuals feature becomes a column-fill + new `event_map` table, not a schema rebuild | — Pending |
| No retry sooner than 10 minutes | Hard-coded 5-minute minimum gap between requests; 10-minute retry gives comfortable margin | — Pending |
| CSV/JSON file exports instead of HTTP API | Zero infrastructure; exports can be committed to the repo as a versioned dataset | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-07-09 after initialization*
