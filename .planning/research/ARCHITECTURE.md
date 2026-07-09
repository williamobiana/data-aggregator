# Architecture Research: FF Harvester

**Domain:** Scheduled financial feed harvesting + Postgres archiving
**Researched:** 2026-07-09
**Confidence:** HIGH — requirements and constraints fully specified in PROJECT.md; patterns well-established for this class of system

---

## System Components

### 1. Scheduler (GitHub Actions cron)
- **Responsibility:** External clock and environment provisioner; owns no state
- **Inputs:** Cron expressions
- **Outputs:** Clean Python environment with secrets injected as env vars

### 2. Fetcher
- **Responsibility:** HTTP GET to the FF endpoint; rate-limit detection; retry policy
- **Inputs:** Target URL, secrets
- **Outputs:** `(ok, body_bytes, http_meta)` or `(ok=False, reason, http_status)`
- **Key:** Detects rate-limit via `Content-Type: text/html` / `<!DOCTYPE` sniff (not a 429); waits 10 min, retries once, then gives up

### 3. Raw Snapshot Writer
- **Responsibility:** Unconditional persistence of full JSON payload before any processing
- **Inputs:** Raw bytes + HTTP metadata
- **Outputs:** Row in `raw_snapshots` (fetched_at, http_status, content_type, payload jsonb, byte_size)
- **Critical:** Run aborts if this insert fails — the safety net must always be written first

### 4. Validator
- **Responsibility:** Structural validation of payload before parsing; catches URL/format changes early
- **Inputs:** Raw bytes
- **Outputs:** Validated list of raw event dicts, or raises `ValidationError`
- **Pure function:** no side effects; fully testable with fixture JSON

### 5. Parser / Transformer
- **Responsibility:** Raw dicts → typed `ParsedEvent` objects; most complex component
- **Inputs:** Validated raw dicts
- **Outputs:** List of ParsedEvents with event_key, UTC timestamps, numeric value + unit decomposition, major-currency filter applied
- **Key:** Parse failures on forecast/previous are warnings only — event is never dropped; raw string always preserved

### 6. Upserter
- **Responsibility:** `INSERT ... ON CONFLICT (event_key) DO UPDATE` against `events` table
- **Inputs:** List of ParsedEvents, DB connection
- **Outputs:** `(inserted=N, updated=M)`; runs in a single transaction

### 7. Fetch Logger
- **Responsibility:** One `fetch_log` row per attempt (including failures and retries)
- **Key:** Uses a separate connection so a rollback of the upsert transaction doesn't swallow the failure log

### 8. Export Pipeline
- **Responsibility:** Dump `events` table to `events.csv` and `events.json`
- **Runs as:** Separate GitHub Actions job on Sunday cron or manual dispatch; read-only

---

## Data Flow

```
GitHub Actions cron
    → Fetcher [HTTP GET, rate-limit detect, retry]
        → Raw Snapshot Writer [write raw_snapshots — ABORT if fails]
        → Validator [structural check — classify as rate_limited or schema_changed if fails]
            → Parser [normalize titles, filter to 8 currencies, UTC conversion, parse values]
                → Upserter [ON CONFLICT upsert → events table]
        → Fetch Logger [fetch_log row — always, success or failure, separate connection]

Sunday 01:00 UTC cron / manual dispatch
    → Export Pipeline [REPEATABLE READ → events.csv + events.json → atomic rename]
    → (optional) commit exports to repo
```

---

## Suggested Build Order

1. **DB schema + migrations** — all other components depend on these tables existing; reserve `actual_*` columns now
2. **Value parser with unit tests** — PROJECT.md explicitly requires this before any other code; pure function, no deps; test DST and edge cases here
3. **Validator** — pure function; test against good/bad/HTML payloads
4. **Fetcher** — simple HTTP wrapper; rate-limit detection must be correct from day one
5. **Raw Snapshot Writer** — first real DB round-trip; validates connection path end-to-end
6. **Parser / Transformer** — integrates the already-tested value parser; unit-testable with fixture dicts
7. **Upserter** — test the ON CONFLICT path explicitly; verify idempotent re-run produces identical rows
8. **Fetch Logger** — wire last; verify failure paths produce log rows
9. **GitHub Actions harvest workflow** — wires 4–8 into a runnable scheduled job with secrets; includes Saturday sweep
10. **Export Pipeline** — separate job; develop and test independently after harvest is stable
11. **AWS RDS provisioning + security hardening** — provision once schema is stable; configure parameter group and security group

---

## Key Design Patterns

| Pattern | How It Applies |
|---------|---------------|
| **Raw-first archiving** | `raw_snapshots` write is a precondition for all processing. Enables full re-parse from history if bugs found. |
| **Deterministic upsert key** | `sha1(country\|normalized_title\|iso_year_week)` — same event, same key, every time. Survives mid-week reschedules. |
| **Parse failures as warnings** | Never drop an event for a parsing edge case; preserve raw strings, NULL numeric fields. |
| **Single-transaction batch upsert** | Partial writes are impossible. Network drop mid-batch rolls back cleanly. |
| **Separate fetch_log connection** | Upsert rollback cannot swallow the failure log entry — both must be visible independently. |
| **fetch_log as operational dashboard** | `SELECT outcome, COUNT(*) FROM fetch_log WHERE attempted_at > now() - interval '7 days' GROUP BY outcome` — no external monitoring tool needed for MVP. |
| **Content-type sniffing** | FF returns HTML on rate-limit, not a 429. Must detect this explicitly or get a confusing JSONDecodeError. |
| **Schema pre-allocation** | `actual_*` columns exist now, stay NULL, require no future migration when actuals feature ships. |
| **UTC normalization at parse time** | All timestamps converted to UTC before any DB write. ISO week computed from UTC timestamp only. |

---

## Fault Tolerance Matrix

| Failure | Detection | Behavior |
|---------|-----------|----------|
| Network timeout | `requests` exception | Log failure row, exit; next run recovers |
| Rate limit (HTML response) | Content-Type / body sniff | Wait 10 min, retry once; log as `rate_limited` |
| Invalid JSON | JSONDecodeError | Log as `parse_error`, exit; re-parse from raw_snapshots |
| Validation failure | ValidationError | Log as `schema_changed`, exit; alert |
| Parse warning (value field) | per-field try/except | NULL field, event continues; logged as warning |
| DB snapshot insert failure | psycopg exception | Abort run; no orphaned un-archived data |
| DB upsert failure | psycopg exception | Rollback, log to fetch_log; next run re-runs cleanly |
| Duplicate run | ON CONFLICT | Idempotent — row updated in place, no harm |
| Export/fetch overlap | REPEATABLE READ txn | Export sees a consistent snapshot regardless of in-progress writes |

---

## File Layout

```
ff-harvester/
├─ src/
│  ├─ harvest.py     # Fetcher → Validator → Raw Snapshot Writer → Parser → Upserter → Fetch Logger
│  ├─ parse.py       # Value parser (pure function) — unit tested in isolation
│  └─ export.py      # Export pipeline — reads events table, writes CSV/JSON
├─ tests/
│  ├─ fixtures/
│  │  ├─ sample_live.json       # real feed payload
│  │  └─ sample_edge_cases.json # blank forecasts, <0.1%, K/M/B suffixes
│  └─ test_parse.py
├─ migrations/
│  └─ 001.sql
├─ requirements.in
├─ requirements.txt  # generated by pip-compile
└─ .github/
   └─ workflows/
      ├─ harvest.yml   # 4×/day + Saturday sweep
      └─ export.yml    # Sunday export
```

---

## Sources

- Project scope document: `.planning/PROJECT.md` (primary)
- PostgreSQL `ON CONFLICT DO UPDATE` documentation
- GitHub Actions scheduled events documentation
- Python psycopg3 connection context manager patterns
- Standard ETL "raw-first" pipeline architecture patterns
