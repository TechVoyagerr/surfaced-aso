<div align="center">

# Surfaced

**An App Store Optimization analytics platform.**

![Python](https://img.shields.io/badge/Python-3.12-3776ab)
![FastAPI](https://img.shields.io/badge/FastAPI-data_plane-009688)
![Postgres](https://img.shields.io/badge/PostgreSQL_16-TimescaleDB-336791)
![Next.js](https://img.shields.io/badge/Next.js-dashboard-000000)

> Commercial product. **The source code is private.**
> This repository is an architecture write-up.

</div>

---

## What it is

Surfaced is an iOS-first ASO platform. It tracks where an app ranks for the keywords that matter, watches what competitors change in their metadata, reads incoming reviews, and turns all of that into a ranked list of keyword opportunities worth acting on.

The product problem underneath it is that Apple publishes almost nothing useful. There is no absolute search volume, no keyword difficulty number, and no ranking API. Everything has to be inferred from observed evidence, which shapes the whole architecture: collect evidence continuously, store it as a time series, and keep the scoring heuristics strictly separate from and versioned against the evidence they read.

## Architecture

```
                    Apple endpoints, autocomplete, SERP
                                   |
        +--------------------------+--------------------------+
        |                          |                          |
   rank_tracker          reviews_fetcher          metadata_snapshotter
        |                          |                          |
        +--------------------------+--------------------------+
                                   |
                        crawl_runs orchestration
                                   |
                    PostgreSQL 16 + TimescaleDB
              (rankings is a hypertable on captured_at)
                                   |
                  +----------------+----------------+
                  |                                 |
          treasure_miner                    FastAPI data plane
      (Keyword Treasure Engine)         19 endpoints, OpenAPI contract
                  |                                 |
                  +----------------+----------------+
                                   |
                    +--------------+--------------+
                    |              |              |
              Next.js web      MCP server      Swift Mac app
```

### Ingestion workers

Three sweeps run under one orchestrated crawl, each recording into a `crawl_runs` row so a partial or failed sweep is visible rather than silent:

| Worker | What it collects |
|---|---|
| `rank_tracker` | Position of each tracked app for each tracked keyword, per country, appended as a time series |
| `reviews_fetcher` | Incoming reviews for sentiment and theme extraction |
| `metadata_snapshotter` | Point-in-time snapshots of title, subtitle, and description, so competitor changes are diffable after the fact |
| `treasure_miner` | Mines autocomplete suggestions and SERP snapshots, then scores keyword opportunities |

The worker entrypoint runs either as a one-shot crawl or as a long-lived asyncio scheduler on a configurable interval, and every collector has an offline mode backed by bundled fixtures so the entire pipeline can be exercised end to end without touching Apple.

### Storage

PostgreSQL 16 with TimescaleDB as the primary target. The schema is applied through numbered SQL migrations and covers apps, keywords, tracked keywords, rankings, metadata snapshots, metadata drafts, reviews, alerts, crawl runs, AI generations, autocomplete suggestions, SERP snapshots and results, keyword treasures, and external keyword metrics.

`rankings` is the time series table and is promoted to a TimescaleDB hypertable on `captured_at`. The migration does this defensively: it attempts the extension and the `create_hypertable` call, and falls back to a plain PostgreSQL table with a notice if either is unavailable, so a developer without TimescaleDB still gets a working database. Redis is provisioned in the Compose stack and health checked by the worker process when `REDIS_URL` is configured, reserved for queue leases and caching.

Data access is SQLAlchemy 2.0 with `psycopg` 3. SQLite is supported as the local fixture-backed target, which is how the whole stack runs on a laptop with no containers.

### The Keyword Treasure Engine

The scoring module is deliberately side effect free. It has no database access and no network access, so the API, the worker, and the tests run exactly the same math. The observed evidence, autocomplete suggestion position and SERP snapshot, is kept separate from the small versioned heuristics that rank opportunities, and the model version is stamped on every result so a scoring change is attributable rather than a silent rewrite of history.

### API

A FastAPI data plane, 19 endpoints, with a checked-in `openapi.yaml` treated as the contract rather than as generated output. Apps, keywords, rankings, reviews, metadata and metadata drafts, treasures and treasure refresh, keyword suggestions, keyword metrics, SERP, metadata validation, AI metadata and review summarization, the external connector sync and status, and a dashboard summary. There is a contract test that holds the implementation to the checked-in spec, and response fields carry backward-compatible aliases where the schema has moved, so an older client does not break on a rename.

### Clients

A Next.js dashboard with Tailwind and Recharts covering keywords, rankings, competitors, reviews, metadata, treasures, and settings. Alongside it, an MCP stdio server that exposes the same API to an AI assistant, and a Swift Mac app.

## Tech stack

| Layer | Technology |
|---|---|
| Language | Python 3.12 |
| API | FastAPI, Pydantic 2, Uvicorn |
| ORM and migrations | SQLAlchemy 2.0, psycopg 3, numbered SQL migrations |
| Database | PostgreSQL 16 with TimescaleDB, SQLite for local fixtures |
| Cache and queue | Redis 7 |
| HTTP client | httpx, async collectors |
| Workers | asyncio scheduler, offline fixture mode |
| Testing | pytest, pytest-asyncio |
| Dashboard | Next.js 14, React 18, TypeScript, Tailwind, Recharts |
| Other clients | MCP stdio server, Swift Mac app |
| Local stack | Docker Compose, uv, Make targets |

## Testing

pytest, covering the OpenAPI contract against the running app, the external connector against a fake MCP server, MCP stdio transport, metadata validation, rank parsing, and the treasure scoring heuristics. The pure scoring module and the fixture-backed offline mode are what make the data pipeline testable without hitting Apple at all.

```bash
make migrate && make seed && make api
pytest
```

---

Source code is private. I am happy to do a full code walkthrough in an interview.
