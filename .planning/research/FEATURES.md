# Features Research: FF Harvester

**Domain:** Scheduled financial data harvesting + Postgres archiving
**Researched:** 2026-07-09
**Confidence:** MEDIUM — synthesized from project scope document and domain knowledge of data pipeline tooling

---

## Table Stakes

| Feature | Why It's Must-Have | Complexity |
|---------|-------------------|------------|
| **Scheduled fetch on cron** | Without reliable triggering (4×/day + Saturday sweep), weeks will be missed and the archive will have gaps that can never be backfilled | Low |
| **Raw snapshot storage before parsing** | If parsing logic has a bug, you need the original payload to replay. Skipping this makes data loss permanent and unrecoverable | Low |
| **Idempotent upsert with deterministic key** | Running the same fetch twice must not create duplicate rows. The sha1(country|title|iso-year-week) key guarantees exactly-once semantics across retries and overlapping schedule runs | Medium |
| **UTC timestamp normalization on ingest** | Financial events from FF come with ambiguous local times. Storing local time makes querying across currency regions incorrect and consumer code fragile | Low |
| **Major-currency filter (8 currencies)** | The feed contains minor currencies that add noise and volume without value for the target use case. Filtering at ingest keeps the archive focused and the export files small | Low |
| **fetch_log per attempt** | Without a log row per attempt (outcome, HTTP status, event count), you cannot tell whether a silence is "no events this week" or "the job silently failed" — the archive looks complete when it isn't | Low |
| **Rate-limit detection and graceful retry** | The FF feed returns HTML when rate-limited rather than a 4xx/5xx. Without detecting this specifically, the tool will silently corrupt data (HTML stored as if it were JSON) or crash | Medium |
| **Numeric value parser with raw-string fallback** | Forecast/previous values come as display strings (e.g. `<0.1%`, `2.3K`, `1.2B`). Downstream consumers need parseable numbers. Parse failures must never drop the event — raw string is the safety net | Medium |
| **Schema with actual_* columns reserved (NULL)** | The actuals feature is explicitly deferred but the column space must exist at creation time. Retrofitting nullable columns into a production table is painful | Low |
| **CSV + JSON export on demand and weekly schedule** | The primary output channel for downstream consumers. Without exports, the archive is only accessible by direct DB query — unusable as a data product | Low |
| **SSL-only RDS access restricted to runner egress** | Credentials in GitHub Actions secrets + SSL + IP restriction is the minimum secure posture for a DB not on the public internet | Medium |
| **Value parser unit tests against real feed samples** | The parser is the most failure-prone logic in the system. Edge cases (`<0.1%`, blank, K/M/B suffix, negative) will appear in production data; untested parsers will silently produce wrong numbers | Medium |

---

## Differentiators

| Feature | Value Proposition | Complexity |
|---------|-------------------|------------|
| **Saturday 23:00 UTC final sweep** | Most archivers run on uniform schedules and miss the last-minute state before weekly rollover. An explicit late-Saturday sweep captures the closing week's final forecast/previous before the feed flips | Low |
| **Separate raw_snapshots table (full payload + metadata)** | Most pipeline tools discard raw input after parsing. Keeping the full JSON payload with ingest timestamp lets you replay any week from scratch and audit feed format changes over time | Low |
| **Week-scoped event key (not timestamp-scoped)** | Rescheduled events within a week update the existing row rather than duplicating it. Tools that key on exact timestamp accumulate phantom duplicates for every mid-week reschedule | Medium |
| **Strict feed-format validation (detect HTML payloads)** | Catching the "got HTML instead of JSON" failure mode early, before processing, prevents silent corruption. Generic HTTP clients that just check status codes miss this entirely | Low |
| **Export committed to repo as versioned dataset** | CSV/JSON exports committed to git give consumers a versioned, diffable historical record — no DB access needed | Low |
| **Parsing stores both raw and parsed form** | Consumers can see the original display string alongside the parsed float+unit. Makes the dataset self-documenting and allows consumer-side re-parsing if units are ambiguous | Low |
| **event_key stability across reschedules** | Because the key is sha1(country|normalized_title|iso-year-week), minor title rewording is partially handled by normalization without needing the deferred event_map alias table | Medium |

---

## Anti-Features (Don't Build in MVP)

| Anti-Feature | Why to Defer | What to Do Instead |
|--------------|-------------|-------------------|
| **Collecting actual results** | Requires a separate data source (FMP/FRED), new fetch pipeline, matching logic, and likely a different schedule. `actual_*` columns are reserved but NULL is the correct MVP state | Reserve columns now; fill in a later phase |
| **HTTP API / REST endpoint** | Adds infrastructure (server, auth, rate limiting, versioning) disproportionate to an internal data product | File-based exports (CSV/JSON) serve consumers without infrastructure cost |
| **Admin UI or dashboard** | Frontend work + infrastructure cost; fetch_log table is the monitoring surface | Query fetch_log directly |
| **Next-week probe** | The endpoint may not exist reliably; MVP success metric doesn't require it; probing risks rate-limit burn | Defer; test manually once the core schedule is stable |
| **event_map alias table** | Title rewording matching across weeks is a data quality concern, not a collection concern | Defer; normalized_title in the key handles obvious cases |
| **Sub-hourly or real-time harvesting** | Feed updates at most a few times per day; 4×/day is already over-sampling | 4×/day cron is sufficient |
| **OAuth or user authentication** | No web interface exists; credentials are GitHub Actions secrets | Credentials via GitHub Actions secrets |
| **Multi-feed support** | Adding other calendar sources multiplies parsing complexity before the core pipeline is proven | Prove FF first; generalize feed abstraction later |
| **Alerting/PagerDuty integration** | Not required for two-week success metric; email-on-failure GitHub Actions is sufficient | Add after the pipeline proves stable |
| **Backfill of historical weeks** | No machine-readable source for past FF data exists | Archive starts from first run; pre-MVP data is unrecoverable by design |

---

## Feature Dependency Chain

```
cron schedule trigger
  → fetch attempt
    → raw snapshot saved (unconditional)
      → feed-format validation (detect HTML payload)
        → rate-limit detection → retry logic
          → major-currency filter
            → UTC normalization
              → numeric value parser (raw string fallback)
                → upsert with deterministic key
                  → fetch_log row written

weekly Sunday export trigger
  → CSV + JSON export from events table
    → (optional) commit exports to repo

RDS provisioning
  → schema migration (raw_snapshots, events, fetch_log tables)
    → actual_* columns reserved NULL

value parser unit tests
  → must pass before any other code runs against real data
```

---

## MVP Build Priority

1. **Value parser with unit tests** — highest failure risk; validate against real samples first
2. **Schema migration** — gates everything; reserve actual_* columns now
3. **Raw snapshot storage** — safety net; no snapshot means permanent data loss if parsing breaks
4. **Fetch + rate-limit detection + graceful retry** — core collection loop
5. **Idempotent upsert** — prevents duplicates on retry or overlapping runs
6. **fetch_log** — observability; without it you cannot tell success from silence
7. **UTC normalization + currency filter** — data hygiene baked into ingest
8. **Saturday 23:00 UTC sweep** — the key differentiator; prevents the specific loss the tool exists to solve
9. **CSV/JSON export** — completes the data product; needed for the success metric

---

## Sources

- Project scope document: `.planning/PROJECT.md` (primary source)
- Domain knowledge: scheduled data pipeline patterns, ETL/ELT idempotency conventions, financial feed behavior
