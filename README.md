<p align="center">
  <img src="https://github.com/user-attachments/assets/9dcf4659-d332-45d7-a0e5-e41a9c3e3696" width="96" alt="Polymarket Backtest logo" />
</p>

# Polymarket Backtest

> This repository documents the architecture and technical design behind [Polymarket Backtest](https://polymarketbacktest.com), a live, paid API product. It is a writeup, not the application source — the codebase is maintained in a private repository since the product is commercial and in production use.

Historical, point-in-time market data for researching and backtesting Polymarket crypto Up/Down strategies.

[Website](https://polymarketbacktest.com) · [API Documentation](https://polymarketbacktest.com/docs)

Polymarket Backtest normalizes recorded prices, spreads, and order-book states into a read-only API. The system consists of a public product site, an authenticated user dashboard, API-key management, usage tracking, subscription handling, and a backtest simulator.

---

## Data Model

Snapshots are stored point-in-time rather than as aggregated candles. This preserves the exact state of a market at a given moment, which avoids lookahead bias when the data is used to simulate strategies against historical order-book conditions.

Supported feeds: BTC, ETH, SOL, XRP, and DOGE, across multiple timeframes. Feeds are addressed through stable aliases, while individual market instances are addressed through exact slugs — this separation allows a caller to query "the current 5-minute BTC market" without needing to track which specific instance is currently active.

---

## Architecture

| Component | Responsibility |
|---|---|
| Next.js application | Marketing pages, documentation, authentication, dashboard, billing flows, and internal server-side requests to the Go API (e.g. for dashboard usage/history views) |
| Go API | Market metadata, snapshot delivery, authentication, pagination, rate limiting, quotas, and usage accounting |
| Supabase | User authentication, API-key records, plan metadata, row-level security, and daily usage data |
| Polar | Checkout, subscription lifecycle, and customer billing portal |
| CSV archive | Append-friendly source files containing normalized market snapshots |
| Docker Compose | Production orchestration for the web and API services |

Since the product's primary offering is the market-data API itself, external API consumers connect to the Go service directly, using their own per-user bearer key, rather than through Next.js. This preserves per-user rate limiting, quota enforcement, and usage accounting at the point where the request actually lands — routing all customer traffic through a single internal credential would collapse per-user metering into one shared identity.

A reverse proxy sits in front of both services and routes by path: `/api/v1/*` is forwarded to the Go service, everything else to Next.js. The Go container is not directly reachable from the internet; it is only reachable through the proxy or over the internal network. Next.js and Go therefore sit side by side behind the proxy rather than Go being fully hidden behind Next.js.

```
Internet
   │
Reverse proxy / API gateway
   ├── /                 → Next.js
   └── /api/v1/*         → Go API
                              ├── Supabase
                              └── CSV archive

Next.js server
   ├── Supabase Auth
   ├── Polar
   └── Internal Go requests (dashboard usage/history views)
```

`INTERNAL_API_KEY` is used only for Next.js's own server-side calls into the Go service — for example, rendering usage or history data in the dashboard. It is not used to proxy customer API traffic; customer requests carry the customer's own bearer key end to end.

The Go service reads snapshot archives directly from disk rather than loading full datasets into memory, and uses reverse-tail reads to serve recent-first pages efficiently from large append-only files.

**Trade-off:** CSV files were chosen for the snapshot archive because the access pattern is append-only, writes are infrequent relative to reads, and it removes the need for a separate time-series database at the current data volume. This works well for the current number of feeds and timeframes; a columnar format or dedicated time-series store would be a more likely fit if the number of tracked assets or the snapshot retention window grows significantly.

---

## Backend Design

The API is a Go HTTP service built with `chi`. Runtime characteristics:

| Feature | Purpose |
|---|---|
| Bearer-token authentication with short-lived plan caching | Avoids a database round-trip on every request while keeping plan changes reflected quickly |
| Per-plan request throttling and monthly quotas | Enforces fair usage across pricing tiers |
| History restrictions by plan | Limits how far back into snapshot history a given plan can query |
| Bounded response sizes and cursor validation | Prevents unbounded queries from degrading service for other users |
| Buffered bandwidth accounting with periodic persistence | Tracks usage without writing to the database on every request |
| Explicit server timeouts, panic recovery, graceful shutdown | Keeps the service stable under load and during deploys |
| JSON error responses with stable, machine-readable codes | Allows API consumers to handle errors programmatically instead of parsing message strings |

### API Surface

Public requests go through the reverse proxy at `/api/v1/*`, which forwards to the Go service's internal `/v1/*` routes.

| Method | Public Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/markets` | Latest metadata for every supported feed |
| `GET` | `/api/v1/markets/{slug}` | Metadata for a feed alias or exact market instance |
| `GET` | `/api/v1/markets/{slug}/snapshots` | Paginated point-in-time snapshots |

All endpoints require an API key:

```bash
curl "https://polymarketbacktest.com/api/v1/markets/btc-updown-5m" \
  -H "Authorization: Bearer mt_live_your_key"
```

Full request parameters, schemas, pagination behavior, and error formats are documented at [polymarketbacktest.com/docs](https://polymarketbacktest.com/docs).

---

## Technology Stack

| Area | Technologies |
|---|---|
| Web | Next.js 16, React 19, TypeScript, Tailwind CSS |
| API | Go 1.23, chi |
| Data & Auth | Supabase, PostgreSQL |
| Billing | Polar |
| Deployment | Docker, Docker Compose |
| Testing | Go test suite, ESLint, Next.js production build |

---

## Repository Structure

```text
.
├── apps/
│   ├── api/                 # Go API and market-data reader
│   │   ├── cmd/api/         # Service entry point
│   │   ├── internal/app/    # HTTP, auth, quota, cache, and market logic
│   │   └── data/            # Local market snapshot archives
│   └── web/                 # Next.js application
│       ├── app/             # App Router pages and route handlers
│       ├── components/      # Product and dashboard UI
│       └── lib/             # Supabase, site, and backtest utilities
├── supabase/                # Database migrations
├── compose.yaml             # Web and API orchestration
└── supabase_schema.sql      # Base database schema
```

---

## Container Deployment

Both services build as separate multi-stage images and run in Docker Compose.

Within the Compose network, both services are reachable at their internal hostnames (`web`, `api`) but only the reverse proxy is exposed to the internet. The proxy routes `/api/v1/*` to the Go container and everything else to the Next.js container, so external API consumers reach Go directly through the proxy while the container itself stays off the public network. Production snapshot storage is mounted into the API container rather than baked into the image, so archives persist independently of deploys.

---

## Verification

The Go service is covered by a test suite (`go test ./...`). The frontend is checked with ESLint and a production build (`next build`) on every change, catching type errors and build failures before deploy.

---

## Security Notes

- Secrets are kept in local or deployment-managed environment files and are not committed to version control.
- Live API keys are only used server-side, or sent directly by API consumers as a bearer token to the Go service.
- Development/test API keys are rejected in production; they are only accepted when the API is explicitly run in a non-production environment with test-key support enabled.
- Supabase service-role credentials bypass row-level security and are only used in server-side contexts.
- The Go container does not publish its port directly to the internet in production; it is reached only through the reverse proxy or the internal Docker network.
- `INTERNAL_API_KEY` is scoped to trusted Next.js → Go server-side calls and is never forwarded to or used by external API consumers.
- API keys are shown in full only at creation time; the dashboard stores and displays a hashed representation afterward rather than the plaintext key.

---

## Disclaimer

This project is an independent research tool and is not affiliated with or endorsed by Polymarket. Historical data and backtest results do not constitute financial advice and do not guarantee future performance.
