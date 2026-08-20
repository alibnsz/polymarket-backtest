<p align="center">
  <img src="https://github.com/user-attachments/assets/9dcf4659-d332-45d7-a0e5-e41a9c3e3696" width="96" alt="Polymarket Backtest logo" />
</p>

# Polymarket Backtest

Historical, point-in-time market data for researching and backtesting Polymarket crypto Up/Down strategies.

[Website](https://polymarketbacktest.com) · [API Documentation](https://polymarketbacktest.com/docs)

> This repository documents the architecture and technical design behind [Polymarket Backtest](https://polymarketbacktest.com), a live, paid API product. It is an architectural writeup, not the application source — the production codebase is maintained privately.

Polymarket Backtest continuously records, normalizes, and indexes high-resolution market snapshots and exposes them through a read-only API for quantitative research and strategy backtesting. The platform includes a public site, authenticated dashboard, API-key management, usage tracking, subscription handling, and a backtest simulator.

---

## Data Model

Snapshots are stored point-in-time rather than reduced to aggregated candles, preserving exact market state at a given moment and avoiding lookahead bias when backtesting. The dataset includes prices, spreads, liquidity, and order-book state.

Supported feeds: BTC, ETH, SOL, XRP, and DOGE across multiple timeframes. Feeds are addressed by stable aliases, individual market instances by exact slugs — so a client can query "the current BTC 5-minute market" without tracking which instance is currently active.

---

## Architecture

| Component | Responsibility |
|---|---|
| Next.js application | Marketing pages, docs, authentication, dashboard, billing |
| Go API | Market metadata, snapshot delivery, auth, pagination, rate limiting, quotas, usage accounting |
| Supabase | Authentication, API-key metadata, plan info, access policies, usage data |
| Polar | Checkout and subscription lifecycle |
| Snapshot archive | Append-friendly storage for normalized historical snapshots |
| Docker Compose | Production service orchestration |

The market-data API is an independent service rather than logic embedded in the web app. External consumers authenticate with their own API key directly against the Go API, so rate limits, quotas, and usage stay tied to the individual customer rather than a single shared credential. Next.js and the Go API sit behind a routing layer as separate services; Next.js also makes limited server-side calls into the API for dashboard functionality, but public API traffic reaches the Go service directly.

The API reads snapshot archives incrementally rather than loading full datasets into memory, using recent-first access patterns to query large append-only files with bounded memory usage.

**Storage trade-off:** flat snapshot archives were chosen because the workload is append-heavy with far more historical reads than writes, which avoids an extra database dependency at the current data volume. A columnar or time-series store would be the natural next step as tracked markets or retention grow.

---

## Backend Design

| Feature | Purpose |
|---|---|
| Bearer-token authentication with short-lived plan caching | Avoids a database round-trip on every request, while still picking up plan or quota changes within seconds rather than requiring a restart |
| Per-plan throttling and quotas | Keeps heavy users from degrading service for others sharing the same infrastructure, and ties usage directly to billing tier |
| Historical access policies | Restricts how far back into snapshot history a request can page based on subscription level, rather than gating access at the account level only |
| Bounded responses with cursor-based pagination | Keeps individual responses small and predictable regardless of how large the underlying snapshot archive grows |
| Buffered usage accounting with periodic persistence | Tracks per-key bandwidth and request counts in memory and flushes on an interval, instead of writing to the database on every request |
| Explicit timeouts, panic recovery, graceful shutdown | Keeps one slow or failing request from taking down the service, and allows in-flight requests to finish cleanly during deploys |
| Machine-readable error codes | Lets API consumers branch on error type programmatically instead of parsing human-readable messages |

---

## API Surface

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/markets` | Latest metadata for supported feeds |
| `GET` | `/api/v1/markets/{slug}` | Metadata for a feed alias or market instance |
| `GET` | `/api/v1/markets/{slug}/snapshots` | Paginated point-in-time snapshots |

```bash
curl "https://polymarketbacktest.com/api/v1/markets/btc-updown-5m" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Full parameters, schemas, and error formats: [API documentation](https://polymarketbacktest.com/docs).

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

## Disclaimer

Polymarket Backtest is an independent research and data tool and is not affiliated with or endorsed by Polymarket. Historical data and backtest results do not constitute financial advice and do not guarantee future performance.
