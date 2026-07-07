# FF Harvester MVP

**What this project does:** Downloads the ForexFactory weekly calendar feed and saves it forever — the event schedule, the forecast, and the previous value — for the 8 major currencies. We are **not** collecting actual results yet. But the database is designed so that when we do add actuals later, we just fill in columns that already exist. No rebuild needed.

---

## 1. What we're building, in one paragraph

A scheduled Python script downloads `https://nfs.faireconomy.media/ff_calendar_thisweek.json` four times a day, checks the data looks right, and saves every major-currency event into a Postgres database. Here's why this matters: the feed only ever shows the current week. Once a week rolls over, the forecasts and previous values it published are gone — you can't go back and get them later. So the whole job of this tool is simple: **never miss a week, never mess up a value.** The end result is a steadily growing `events` table, plus a CSV/JSON export you can download.

---

## 2. What the feed gives us (checked against a real sample)

The feed is a list of objects that look exactly like this:

```json
{
"title":    "German Factory Orders m/m",
"country":  "EUR",
"date":     "2026-07-06T02:00:00-04:00",
"impact":   "Low",
"forecast": "1.1%",
"previous": "-3.8%"
}
```

What each field means and how we handle it:

- `title` — the event name as displayed. Slashes come JSON-escaped (`m\/m`), so we clean those up when parsing. Heads up: ForexFactory sometimes rewords titles over time.
- `country` — actually a currency code. We keep `USD, EUR, GBP, JPY, AUD, NZD, CAD, CHF` and throw away the rest. For those 8, we keep **everything** — storage is basically free, and anything we filter out now can never be recovered later.
- `date` — a timestamp with a timezone offset (looks like US Eastern). We convert to UTC the moment it comes in and never store local time anywhere.
- `impact` — one of `High`, `Medium`, `Low`, or `Holiday`. In the first week we'll double-check what holiday/non-economic values actually show up and log anything unexpected.
- `forecast` and `previous` — display strings. **An empty string is normal and fine** — it just means no consensus was published, which happens a lot for Low-impact events. There is **no actual field** in this feed at all.

Things to keep in mind about the source: roughly 2 downloads per 5 minutes per IP before you get rate-limited; when you're rate-limited you get an HTML page instead of JSON; the feed only shows the current week, rolling over around the weekend; and it's an unofficial feed, so the URL or format could change at any time without warning.

---

## 3. When we download

| Job | Schedule (UTC) | Why |
|---|---|---|
| Regular harvest | `0 0,6,12,18 * * *` (4 times a day) | Catch the new week, forecast changes, and rescheduled events |
| **Saturday final sweep** | `0 23 * * 6` (Saturday 11pm) | Grab the closing week's final state before it disappears — the single most important download of the week |
| Next-week probe (optional) | `30 12 * * *` (once a day) | If early testing confirms a `ff_calendar_nextweek.json` exists, we can capture forecasts days earlier. Same code, just pointed one week ahead |

**When a download fails** (we get HTML instead of JSON, an HTTP error, or an empty list): wait 10 minutes and try once more — never sooner, because there's a hard-coded rule of at least 5 minutes between any two requests. If the retry also fails, give up and wait for the next scheduled run. Alerting costs nothing: a failed GitHub Actions run sends you an email automatically. The only failure worth actually worrying about is the Saturday sweep — if that one fails, rerun it by hand.

---

## 4. The database (3 tables)

**`raw_snapshots`** — every successful download, saved exactly as received. This is the safety net: if we ever discover a parsing bug, we can re-process everything from these. Never skip saving one.

```sql
id          bigserial PRIMARY KEY,
fetched_at  timestamptz NOT NULL,
week_key    date NOT NULL,          -- the Sunday (UTC) of the week this payload covers
week_final  boolean DEFAULT false,
payload     jsonb NOT NULL,
byte_size   int
```

**`events`** — the actual product. One row per real-world event.

```sql
id             bigserial PRIMARY KEY,
event_key      text UNIQUE NOT NULL,   -- sha1(country|normalized_title|iso_week)
title          text NOT NULL,
country        text NOT NULL,
scheduled_at   timestamptz NOT NULL,   -- UTC
impact         text,
forecast_raw   text,                   -- exactly as published: '1.1%', '-14.5B', ''
previous_raw   text,
forecast_num   numeric,                -- parsed number; NULL if blank or unparseable
previous_num   numeric,
unit           text,                   -- 'percent' | 'K' | 'M' | 'B' | 'none'
-- reserved for the future actuals feature — created now, stays NULL for the MVP:
actual_raw     text, actual_num numeric, actual_source text, actual_first_seen_at timestamptz,
first_seen_at  timestamptz NOT NULL,
last_updated_at timestamptz NOT NULL
```

**How we identify an event:** `event_key = sha1(country + '|' + normalized title + '|' + ISO year-week of the scheduled date)`. Normalizing the title means trimming it, collapsing repeated spaces, lowercasing, and fixing escaped slashes. The key is based on the *week*, not the exact timestamp — so if an event gets moved from Tuesday to Wednesday mid-week, we **update** the existing row instead of creating a duplicate. If one payload ever produces two events with the same key (possible for items that repeat within a week), log it loudly; for the MVP it's fine to just keep the latest one, and we'll revisit if that log ever actually fires.

**How updates work:** if the event is new, insert it. If we've seen it before, update the schedule, impact, forecast/previous values, unit, and the last-updated timestamp. A changed forecast is a real revision, so the latest value wins — and if we ever want the older value, it's still sitting in the raw snapshots.

**`fetch_log`** — one row per download attempt: when it ran, which week it targeted, the HTTP status, the outcome (`ok`, `rate_limited`, `parse_error`, `network`, or `empty`), and how many events were seen, added, and updated. This is how we spot gaps and check the system's health.

---

## 5. Turning strings into numbers

One small, pure function: `parse(raw)` takes the display string and returns a number, a unit, and sometimes a flag.

| Input | Number | Unit |
|---|---|---|
| `"1.1%"` | 1.1 | percent |
| `"-13.4"` | -13.4 | none |
| `"-14.5B"` | -14,500,000,000 | B |
| `"215K"` / `"7.6M"` | 215,000 / 7,600,000 | K / M |
| `""` | null | null |
| `"<0.1%"` | 0.1 | percent, flagged as "less than" |
| anything else | null | null, and log a warning |

Ground rules: strip whitespace and commas first; the raw string is always stored no matter what happens; and a parse failure is only ever a warning — it never causes an event to be dropped. **This function gets unit tests before any other code is written**, using the real live sample as the first test fixture.

---

## 6. Stack & layout

GitHub Actions cron → one Python script → an AWS RDS Postgres instance. No servers to run, no framework. The only real running cost is RDS itself — a small instance (e.g. `db.t4g.micro`) is free for the first 12 months on the AWS free tier and cheap after that; this workload is tiny, so the smallest instance is plenty.

```
ff-harvester/
├─ src/harvest.py        # download → validate → parse → upsert (~200 lines)
├─ src/parse.py          # the value parser (pure function)
├─ src/export.py         # dump the events table to CSV/JSON
├─ tests/test_parse.py   # fixtures, including the real live sample
├─ migrations/001.sql
├─ requirements.txt      # requests, psycopg (that's about it)
└─ .github/workflows/harvest.yml   # the three cron schedules
```

The checks inside `harvest.py`, in order: got HTTP 200 → check the content type / first character (if it's HTML, we're rate-limited — stop) → parse the JSON → confirm it's a non-empty list → confirm every item has `title`, `country`, and `date` → keep only the 8 currencies → save the raw snapshot → parse the values → upsert into `events` → write a `fetch_log` row.

One practical note on the database connection: RDS should not be open to the whole internet. The simplest setup for this project is to allow connections only from GitHub Actions (or run the connection through a fixed egress IP / a tiny bastion), require SSL, and keep the credentials in GitHub Actions secrets.

**What "export" means for the MVP:** `export.py` dumps the full `events` table to `events.csv` and `events.json`, both on demand and on a weekly schedule (Sunday, after the Saturday sweep), optionally committing the files back to the repo — a zero-infrastructure way to have a downloadable, versioned dataset. A proper HTTP API is a later project, alongside actuals.

---

## 7. Build order (realistically 2–3 evenings)

1. **Test data first (30 min):** save the real live payload, plus one hand-edited copy with the weird cases (blank forecasts, `<0.1%`, an oddly formatted title), into `tests/`.
2. **Parser + its tests (1–2 hours).**
3. **Database migration + upsert + harvest script (half a day):** run it by hand a few times, look at the rows, and confirm that running it twice in a row adds zero new events — that proves it's safe to rerun.
4. **Schedules + Saturday sweep + failure email (1–2 hours):** then let it run on its own.
5. **Export script (1 hour).**
6. **Week-two checkup (15 min):** look at `fetch_log` for gaps, confirm both weeks are fully there, and confirm the week rollover created new rows instead of overwriting old ones. If it all looks clean, the MVP is done.

---

## 8. What could go wrong (and what we do about it)

| Risk | How we handle it |
|---|---|
| The Saturday sweep fails, losing a week's final state | The 4×/day schedule means at most ~6 hours of late revisions are lost; the failure email prompts a manual rerun |
| The feed's URL or format changes | Strict validation fails loudly instead of saving garbage; raw snapshots mean recovery is a re-parse, not lost data |
| A retry bug gets our IP banned | The 5-minute minimum gap is enforced in code, and there's a maximum of 1 retry per run |
| GitHub Actions runs late or skips a run | Harmless at this frequency; `fetch_log` shows any gaps |
| The same event appears twice because its title was reworded mid-week | Rare within a single week; we log any key collisions; a proper alias map is a later-iteration job |

## 9. When we call it done

Two consecutive weeks running unattended, with: every major-currency event archived with forecast and previous (both the raw string and the parsed number); both Saturday sweeps captured (snapshots marked `week_final=true` exist); reruns proven safe (no duplicates); zero parse warnings on the standard formats; one clean CSV/JSON export produced; and zero rate-limit incidents in `fetch_log`.

## 10. What's already prepared for next time (not built now)

The future actuals feature (pulling actual results from FMP, with FRED as an override, per spec v0.2) will simply fill the four `actual_*` columns that already exist; an `event_map` table gets added then; and "surprise" (actual vs. forecast) becomes computable the day actuals arrive — retroactively for every event we've archived, since FMP supports historical date-range queries. Nothing built in this MVP will need to be redone.