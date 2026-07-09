# Stack Research: FF Harvester

**Project:** ForexFactory Weekly Calendar Harvester
**Researched:** 2026-07-09
**Mode:** Ecosystem — greenfield Python scheduled harvester

---

## Recommended Stack

### HTTP Client

- **Library:** `requests` 2.32.x
- **Why:** The PROJECT.md constraint explicitly calls for `requests`. Beyond that constraint, it remains the right call: zero learning curve, synchronous by default (matching the single-threaded harvest script), JSON parsing built-in via `.json()`, timeout and retry configuration are idiomatic and well-documented. The FF feed is one endpoint hit four times a day — async HTTP (httpx, aiohttp) adds complexity with zero benefit at this call volume.
- **Rate-limit handling:** Wrap in a thin retry decorator that checks `response.headers['Content-Type']` for `text/html` (the FF rate-limit signal), waits 10 minutes, retries once. `requests` makes this trivial.
- **Confidence:** HIGH

### Postgres Driver

- **Library:** `psycopg[binary]` 3.2.x (psycopg3)
- **Why:** psycopg2 is in maintenance mode as of 2024 — no new features, security fixes only. psycopg3 (`psycopg`) is the actively maintained successor from the same author, ships with native binary protocol support, and has first-class `executemany` + `COPY` performance. For a synchronous script (no async needed), the standard `psycopg.connect()` API is drop-in familiar. The `[binary]` extra uses the faster C extension; falls back to pure Python if the C extension is unavailable on the runner.
- **SSL:** `psycopg` accepts `sslmode=require` and `sslrootcert` in the connection string — exactly what RDS SSL enforcement needs.
- **Confidence:** HIGH

### Upsert / Deduplication Key (sha1)

- **Library:** Python stdlib `hashlib` (no extra dependency)
- **Why:** `hashlib.sha1(f"{country}|{normalized_title}|{iso_year_week}".encode()).hexdigest()` is deterministic, fast, and produces a fixed 40-char hex string that fits a `CHAR(40)` column. No third-party dependency needed. SHA-1 collision risk is irrelevant for a deduplication key over a small, controlled string space — this is not a cryptographic use.
- **Confidence:** HIGH

### Value Parser (forecast/previous strings)

- **Library:** Python stdlib `re` + `decimal.Decimal`
- **Why:** The feed strings (`<0.1%`, `1.2K`, `-3.4M`, `2.1B`) are a closed, finite pattern set. A regex dispatch table handles every variant in ~50 lines. `Decimal` avoids float rounding artifacts in stored numeric values. No third-party parsing library (pint, babel) is warranted.
- **Confidence:** HIGH

### Database Migrations

- **Library:** Plain SQL files applied via `psql` in CI
- **Why:** This project has one migration at MVP (create three tables). A full migration framework (Alembic, Flyway, yoyo-migrations) is overkill. A single `migrations/001.sql` file run with `psql $DATABASE_URL < migrations/001.sql` in a one-shot GitHub Actions job is simpler and has zero runtime dependency. Graduate to a migration tool if schema changes become frequent.
- **Confidence:** HIGH

### Testing Framework

- **Library:** `pytest` 8.x
- **Why:** pytest is the de-facto standard for Python unit testing in 2025. Parametrize decorator makes value-parser testing against a fixture table (blank, `<0.1%`, `1.2K`, `-3.4M`, `2.1B`, normal `1.5%`) trivially clean. `pytest-cov` for coverage. No reason to use `unittest` — it requires class-based boilerplate that adds noise without benefit.
- **Confidence:** HIGH

### Environment / Config

- **Library:** `python-dotenv` 1.x (dev only) + GitHub Actions Secrets (production)
- **Why:** `.env` file for local development, GitHub Actions Secrets (`${{ secrets.DATABASE_URL }}` etc.) for CI. No config management framework needed. The `python-dotenv` load is wrapped in a `if not os.getenv("CI"):` guard so it never fires in Actions.
- **Confidence:** HIGH

### CSV / JSON Export

- **Library:** Python stdlib `csv` + `json` modules
- **Why:** Standard library covers both export formats entirely. `csv.DictWriter` produces RFC 4180-compliant output from a list of dicts. `json.dumps` with `default=str` handles `datetime` and `Decimal` serialization. No pandas dependency — it's 35 MB of binary and adds a cold-start penalty to the Actions runner for zero benefit when you're writing a flat table dump.
- **Confidence:** HIGH

### GitHub Actions Runner

- **Runtime:** `ubuntu-latest` (Ubuntu 24.04 as of mid-2025)
- **Python version:** `python-version: "3.12"` via `actions/setup-python@v5`
- **Why 3.12:** Released October 2023, supported until October 2028. Stable, widely compatible. Avoid 3.13 (still accumulating ecosystem compatibility as of mid-2025).
- **Cron syntax:** GitHub Actions cron runs in UTC. Schedule strings:
  ```yaml
  schedule:
    - cron: '0 0 * * *'   # 00:00 UTC daily
    - cron: '0 6 * * *'   # 06:00 UTC daily
    - cron: '0 12 * * *'  # 12:00 UTC daily
    - cron: '0 18 * * *'  # 18:00 UTC daily
    - cron: '0 23 * * 6'  # 23:00 UTC Saturday sweep
  ```
- **Important caveat:** GitHub Actions cron is not guaranteed on the exact minute — typical jitter is 5–15 minutes, occasionally longer during high load. Fine for a 4×/day schedule with 6-hour windows.
- **Confidence:** HIGH

### AWS RDS Postgres

- **Engine version:** PostgreSQL 16.x (latest stable LTS on RDS as of mid-2025)
- **Instance class:** `db.t4g.micro` — ARM-based, free-tier eligible for 12 months, sufficient for a write-light append workload
- **SSL:** Set `rds.force_ssl=1` parameter group. Download the AWS RDS CA bundle and reference it in `sslrootcert`. Connection string: `postgresql://user:pass@host:5432/db?sslmode=verify-full&sslrootcert=/path/to/ca.pem`
- **Network access:** RDS in a VPC with a security group that whitelists GitHub Actions egress IPs. GitHub publishes its meta IP ranges at `https://api.github.com/meta` — the `actions` key. These change; a scheduled Actions job or Lambda can keep the SG current.
- **Confidence:** HIGH (RDS config), MEDIUM (IP whitelisting durability — see Pitfalls)

### Dependency Pinning

- **Tool:** `pip-tools` (`pip-compile`) to generate a locked `requirements.txt` from `requirements.in`
- **Why:** Reproducible builds across developer machines and CI. Lock file committed to the repo. `pip install -r requirements.txt` in Actions.
- **Confidence:** HIGH

---

## Minimal `requirements.in`

```
requests>=2.32
psycopg[binary]>=3.2
python-dotenv>=1.0
pytest>=8.0
pytest-cov>=5.0
pip-tools>=7.0
```

Everything else (hashlib, csv, json, re, decimal) is Python stdlib.

---

## What NOT to Use

| Tool | Why to Avoid |
|------|-------------|
| `httpx` | No benefit over `requests` for a synchronous single-endpoint script |
| `aiohttp` / `asyncio` | Async adds complexity; one HTTP call per run — async is pure overhead |
| `psycopg2` | Maintenance mode since 2024; psycopg3 is the supported successor |
| `asyncpg` | Async-only; wrong paradigm for this synchronous harvest script |
| `SQLAlchemy` | Full ORM is 10x the complexity needed for three-table upserts |
| `pandas` | 35 MB cold-start overhead for CSV/JSON export that stdlib handles in 20 lines |
| `Alembic` | Overkill for a single MVP migration; plain SQL file is sufficient |
| `celery` / `APScheduler` | No in-process scheduler needed — GitHub Actions cron IS the scheduler |
| Python 3.13 | Ecosystem compatibility still settling as of mid-2025; stick to 3.12 |

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| HTTP client | HIGH | `requests` is unambiguously correct for this workload |
| Postgres driver | HIGH | psycopg3 is the settled successor; psycopg2 is deprecated |
| GitHub Actions cron | HIGH | Well-documented, widely used pattern |
| RDS SSL config | HIGH | AWS docs are canonical; verify-full + CA bundle is standard |
| RDS IP whitelisting | MEDIUM | GitHub Actions egress IPs change without warning; needs maintenance plan |
| Value parser approach | HIGH | Stdlib regex + Decimal is the right tool for a closed string set |
| Testing (pytest) | HIGH | No serious alternative in 2025 Python ecosystem |
| Export (csv/json stdlib) | HIGH | stdlib is sufficient; no third-party library needed |

---

## Sources

- psycopg3 official docs: https://www.psycopg.org/psycopg3/docs/
- GitHub Actions scheduled events: https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule
- GitHub Actions meta IP ranges: https://api.github.com/meta
- AWS RDS SSL/TLS documentation: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html
- AWS RDS free tier instance types: https://aws.amazon.com/rds/free/
- pytest documentation: https://docs.pytest.org/en/stable/
- requests documentation: https://requests.readthedocs.io/
