# Pitfalls Research: FF Harvester

**Domain:** Scheduled financial feed harvesting — Python + GitHub Actions + AWS RDS Postgres
**Researched:** 2026-07-09
**Confidence:** HIGH

---

## Critical Pitfalls

### Pitfall 1: Missing the Saturday Rollover — "The Week You Lost Forever"

The single most dangerous failure for this project. The FF feed only shows the current week. If the Saturday 23:00 UTC sweep is skipped for any reason, all forecast and previous values for that week are gone permanently. No recovery possible from any source.

- **Warning signs:** `fetch_log` gap of more than 7 hours near a Saturday night UTC boundary; any Saturday row with `outcome != ok`; ISO week number in consecutive successful fetches jumps by more than 1
- **Prevention:** Treat the Saturday 23:00 sweep as a distinct monitored job. Add a Sunday 01:00 UTC monitoring step that checks `fetch_log` for whether a Saturday 23:00 run exists; if missing, trigger immediate re-fetch via `workflow_dispatch`. Store ISO week number of last successful fetch; assert continuity on every run.
- **Phase:** Harvester core + alerting, Phase 1 (before any unattended week runs)

---

### Pitfall 2: Non-Idempotent Upserts — Phantom Duplicates on Retry

`INSERT ... ON CONFLICT DO UPDATE` is only safe if every mutable column is listed in the `SET` clause. Omitting `forecast_raw` or `previous_raw` from the `UPDATE` silently keeps stale values on retry after a partial failure.

- **Warning signs:** Row count in `events` grows faster than unique `event_key` count; column values that can't be right on re-inspection; `fetch_log` INSERT count diverges from expectations
- **Prevention:** Write the upsert listing every column explicitly in `DO UPDATE SET forecast_raw = EXCLUDED.forecast_raw, ...`. Test idempotency: run the same payload twice, assert row count and values are identical after both runs.
- **Phase:** Schema migration + unit tests, Phase 1

---

### Pitfall 3: Storing Local Time Instead of UTC — Silent Timestamp Drift

The FF feed delivers timestamps in US/Eastern timezone (offset shifts with DST). Storing the raw string or naively converting without anchoring to UTC causes events to appear at the wrong time. The drift is 1 hour twice per year and is never obvious from the data itself.

- **Warning signs:** A Monday 08:30 New York event appears at 13:30 UTC in March but 12:30 UTC in November; events cluster at wrong hours around DST transition weekends
- **Prevention:** Parse the FF timestamp with explicit `zoneinfo.ZoneInfo('America/New_York')`, then `.astimezone(timezone.utc)` before any DB write. Store `TIMESTAMPTZ` in Postgres, never `TIMESTAMP WITHOUT TIME ZONE`. Unit-test against real DST-transition samples.
- **Phase:** Parser unit tests, Phase 1 (before any live run)

---

### Pitfall 4: Treating an HTML Rate-Limit Response as Success — Silent Data Gaps

When the FF feed rate-limits a request it returns HTTP 200 with an HTML body, not a 4xx/5xx error. Code that checks only `response.status_code == 200` treats this as a successful empty response, logs nothing useful, and produces a gap in the archive with no indication of why.

- **Warning signs:** `fetch_log` rows with `event_count = 0` and `outcome = ok`; `response.text` starts with `<!DOCTYPE` or `<html>`; archive gaps with no logged error
- **Prevention:** After every request, check that the body parses as valid JSON and matches the expected feed schema. If validation fails, classify as `rate_limited` (HTML body) or `schema_changed` (unexpected JSON shape), log the raw response, and abort without writing to `events`.
- **Phase:** Harvester core — HTTP client layer, Phase 1

---

### Pitfall 5: GitHub Actions Cron Skips During High Platform Load

GitHub Actions `schedule` cron jobs are explicitly not guaranteed to fire at the scheduled time. During high platform load they can be delayed 15–60 minutes or skipped entirely. A skipped Saturday 23:00 UTC run is irreversible data loss.

- **Warning signs:** `fetch_log` timestamps arriving 20–40 minutes past the hour; gap of more than 2 hours between consecutive `fetch_log` rows; GitHub status page incident during a critical window
- **Prevention:** Add a Sunday 01:00 UTC monitoring job that checks `fetch_log` for the Saturday 23:00 run; if missing, trigger re-fetch via `workflow_dispatch`. Keep the Saturday sweep in its own dedicated workflow file to reduce queue pressure.
- **Phase:** Workflow setup, Phase 1

---

### Pitfall 6: RDS Connection Exhaustion

`db.t4g.micro` has a default `max_connections` of ~87. Running the script repeatedly during debugging can exhaust connections. A failed connection during a production run is a missed fetch window.

- **Warning signs:** `psycopg.OperationalError: FATAL: remaining connection slots are reserved`; RDS CloudWatch `DatabaseConnections` near the instance limit; connections that don't close on unhandled exceptions
- **Prevention:** Always use a context manager (`with psycopg.connect(...) as conn:`). Set `idle_in_transaction_session_timeout = '30s'` in the parameter group. Open exactly one connection per script invocation.
- **Phase:** RDS provisioning, Phase 1

---

### Pitfall 7: Event Key Collisions from Non-Deterministic Title Normalization

`event_key = sha1(country|normalized_title|iso_year_week)` is only collision-safe if `normalized_title` is produced by a frozen, deterministic function. If normalization changes between code versions, the same real-world event produces two different keys — creating phantom duplicates.

- **Warning signs:** Two rows in `events` with the same `country`, `title`, and ISO week but different `event_key`; row count grows faster than expected after a code change to the normalization function
- **Prevention:** Define and freeze the normalization function before any data is written to production. Document it explicitly. If it must ever change, include a one-time migration that recomputes all existing `event_key` values before resuming. Test against edge-case titles: slashes, parentheses, Unicode apostrophes, trailing periods.
- **Phase:** Schema + parser unit tests, Phase 1 (before first production insert)

---

### Pitfall 8: Forecast/Previous Parser Silently Losing Data

A parser that raises an exception on unrecognized formats and is caught by a bare `except: pass` silently stores `NULL` for values it failed to handle, making the archive look sparse when the raw value was actually present.

- **Warning signs:** `forecast_num` is NULL but `forecast_raw` is non-empty for a substantial fraction of rows; any new suffix or format in the feed produces a spike in NULL parsed values
- **Prevention:** The parser must return `(value, unit, parse_error)` — never raise. Log `parse_error` as a warning without dropping the row. Unit-test every known format: `""`, `"-"`, `"0.3%"`, `"<0.1%"`, `"1.2K"`, `"500M"`, `"1.5B"`, negative numbers. Post-run assertion: if more than 10% of rows with non-empty `forecast_raw` have NULL `forecast_num`, alert.
- **Phase:** Parser unit tests, Phase 1

---

### Pitfall 9: Raw Snapshot Storage Failing Silently

If snapshot writes fail silently (DB error caught and suppressed), the safety net disappears without anyone noticing — until a parsing bug is discovered and there is nothing to replay from.

- **Warning signs:** `raw_snapshots` row count is less than `fetch_log` successful-fetch count; any DB error during snapshot insert that is caught and suppressed
- **Prevention:** Write the snapshot insert inside the same transaction as the `fetch_log` row — if the snapshot fails, the fetch_log row must also fail, making the gap visible. Assert on every run that the snapshot was written before proceeding to parse.
- **Phase:** Harvester core, Phase 1

---

### Pitfall 10: RDS Security Group Drift — GitHub Actions IP Ranges Change

GitHub Actions egress IP ranges change periodically. Hardcoding a static IP list in the security group means it drifts out of date and production runs start failing — or worse, an engineer opens the security group to `0.0.0.0/0` as a "quick fix."

- **Warning signs:** GitHub Actions run fails with connection refused after a period of working fine, with no code changes; the GitHub meta API `actions` IP list has entries not in the RDS security group
- **Prevention:** Add a weekly automated workflow that fetches the GitHub meta API, extracts `actions` IP ranges, and compares against current security group rules — alerting if they diverge. Never use `0.0.0.0/0` as a fallback.
- **Phase:** RDS/infrastructure Phase 1 (setup); IP-drift monitoring can be Phase 2

---

## Moderate Pitfalls

### Pitfall 11: Feed URL or Schema Changes With No Notice

The FF feed is unofficial. The URL and field names can change without warning. Code that hardcodes field names without validation will silently process malformed data or crash.

- **Warning signs:** Expected top-level keys (`title`, `country`, `date`, `impact`, `forecast`, `previous`) absent from response; `event_count` drops to an implausibly low number
- **Prevention:** Validate JSON schema on every fetch (required keys present, types correct). If validation fails, classify as `schema_changed` in `fetch_log` and do not write to `events`.
- **Phase:** Harvester core, Phase 1

---

### Pitfall 12: ISO Week Number Computed in Local Time

Python's `datetime.isocalendar()` returns the ISO week for whatever timezone the datetime object is in. A Saturday event that is Saturday in New York but Sunday UTC will be assigned to the wrong ISO week, breaking the event key.

- **Warning signs:** Events from the last day of a week appearing in the following week's key; `event_key` for a known event changes between the Saturday sweep and UTC Sunday morning
- **Prevention:** Always compute the ISO year-week from the UTC-normalized timestamp. Rule: normalize to UTC first, then `.isocalendar()`. Test against a DST-boundary Saturday sample.
- **Phase:** Parser unit tests, Phase 1 (same test suite as Pitfall 3)

---

### Pitfall 13: Export Job Overlapping an In-Progress Fetch

If the Sunday export and a regular fetch run overlap, the export may read a partially-written `events` snapshot.

- **Warning signs:** Export CSV row count lower than previous export by more than expected weekly increment
- **Prevention:** Schedule the export at 01:00 UTC Sunday (well after Saturday 23:00 UTC sweep). Wrap export query in a `REPEATABLE READ` transaction. Write exports to a temp path, then rename atomically.
- **Phase:** Export feature, Phase 2; but transaction isolation choice is Phase 1 schema work

---

## Minor Pitfalls

### Pitfall 14: Logging Secrets in GitHub Actions Output

Postgres credentials in DSN strings can appear in exception tracebacks, leaking them in Actions run logs.

- **Prevention:** Pass connection parameters individually (host, user, password, dbname) to `psycopg.connect()` rather than as a single DSN string. Never log exception objects from connection errors directly.
- **Phase:** Harvester core, Phase 1

---

### Pitfall 15: Application-Side Timestamps Creating Clock Skew

Using `datetime.utcnow()` in Python for `fetch_log.fetched_at` vs. `CURRENT_TIMESTAMP` set by the DB server can create ordering anomalies in queries that join `fetch_log` to `raw_snapshots`.

- **Prevention:** Use `CURRENT_TIMESTAMP` (DB-server time) for all timestamp columns recording when a DB operation happened. Use application-generated timestamps only for values representing external event times.
- **Phase:** Schema definition, Phase 1

---

## Phase-Specific Warning Map

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Schema migration | P2 (non-idempotent upsert), P7 (event key normalization) | Test idempotency before any live run |
| Parser implementation | P8 (silent value loss), P3 (DST/UTC), P12 (ISO week in local time) | Unit tests run before harvester code |
| HTTP client | P4 (HTML-as-success), P11 (feed schema change) | Validate JSON body on every fetch |
| GitHub Actions cron | P1 (missed Saturday sweep), P5 (cron skip) | Dedicated sweep workflow + monitoring job |
| RDS provisioning | P6 (connection exhaustion), P10 (IP drift) | Parameter group + IP-drift check workflow |
| Raw snapshot save | P9 (silent snapshot failure) | Snapshot in same transaction as fetch_log |
| Export feature | P13 (export/fetch overlap) | Export at 01:00 UTC Sunday + atomic rename |
| Code review | P14 (secret leakage), P15 (clock skew) | Code review checklist |

---

## Sources

- `.planning/PROJECT.md` — feed behavior, rate limiting, key design decisions, constraints
- GitHub Actions schedule documentation — cron jobs are not guaranteed to fire at exact scheduled time
- PostgreSQL `ON CONFLICT DO UPDATE` semantics — explicit column list required in `UPDATE SET`
- Python `zoneinfo` DST handling — `America/New_York` transitions second Sunday March, first Sunday November
- AWS RDS `db.t4g.micro` default `max_connections` — ~87
- GitHub meta API for Actions egress IPs — `https://api.github.com/meta`, `actions` key
