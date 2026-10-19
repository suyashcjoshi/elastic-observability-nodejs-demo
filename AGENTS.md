# AGENTS.md

Skyward for Business is a demo Node.js app. This is the **`fixed`** branch: it has **no intentional bugs**. The `main` branch is the deliberately broken version used for the Elastic Observability demo.

Do **NOT** reintroduce any of the `main` branch bugs:

1. Sequential partner `await`s in `/api/search` (use `Promise.allSettled` fan-out)
2. O(n²) dedupe (use the `Map`-based dedupe)
3. No outbound timeout on partner calls (keep `AbortSignal.timeout(1500)`)
4. Missing `await` on `pool.query()` in `/api/chat`

There are no `CHAOS_*` / `ENABLE_CHAOS` flags on this branch.

## Run

    cp -n .env.example .env
    npm install
    npm run dev          # starts postgres (docker), seeds, starts partners + app
    npm run verify       # checks /health, /api/search, /api/insights, /api/chat, /api/book; exits 0 on success
    npm run stop:all

## Ports

    3000  app
    4001  PuffinAir (PF)
    4002  GooseJet (GJ)
    4003  Pelican Air (PA)
    4004  Penguin Air (PN) — slow (5.2 s) and flaky (15% 503)
    5432  postgres

## API endpoints

    GET  /health                                              — liveness check
    GET  /api/search?origin&destination&date&user&sort&purpose — fan-out to 4 partners, dedupe, return top 50
    GET  /api/insights?origin&destination&user               — avg price + pct change vs prev month + recent searches
    POST /api/chat     { message, user }                     — keyword-based canned reply
    POST /api/book     { flight_id, user }                   — confirm with partner (8 s timeout, one retry); 502 if the partner fails

    Each partner also exposes:
    GET /confirm?flight_id=<id>  — confirmation endpoint called by /api/book

## Elastic

Set `OTEL_EXPORTER_OTLP_ENDPOINT` and `OTEL_EXPORTER_OTLP_HEADERS` in `.env`,
then `npm run restart:elastic`. Never commit `.env`.

## Repo

https://github.com/suyashcjoshi/elastic-observability-nodejs-demo

## Deploy

    docker compose up -d      # db, partners, app

## Conventions

- ESM only (`"type": "module"` in package.json). No CommonJS `require()` in `src/`.
- No OpenTelemetry code in `src/` — EDOT is loaded via `--import @elastic/opentelemetry-node`.
- Node >= 20.6 (required for `--env-file` and `--import`).
- Run `npm run verify` after any change to confirm the golden path still works.
- Run `npm test` to run the smoke test suite.
- Every response carries `X-Request-Id: req_<12hex>` — generated per-request by request ID middleware.
