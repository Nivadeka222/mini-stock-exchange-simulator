# Mini Stock Exchange Simulator

A low-latency exchange simulator: a **C++ matching engine** (price-time priority order book), a **Go backend** (REST + WebSocket gateway, gRPC client), and a **Next.js dashboard** (live order book, trades, and stats) — three services, each doing one job, talking to each other the way real trading infrastructure does.

**Live demo:** [stock-exchange-sim.vercel.app](https://stock-exchange-sim.vercel.app)
**Backend API:** [mini-stock-exchange-simulator.onrender.com](https://mini-stock-exchange-simulator.onrender.com) ([`/health`](https://mini-stock-exchange-simulator.onrender.com/health))

> Hosted on free tiers — the backend spins down after 15 minutes idle, so the first request after a while can take 30-60s to wake up. Give it a minute if the book looks empty on first load.

---

## What it does

- Accepts buy/sell limit orders over a REST API
- Matches them in a C++ order book using **price-time priority** — best price first, and FIFO between orders at the same price, same rule real exchanges use
- Supports partial fills: an incoming order can eat into a resting order without fully clearing it
- Streams every order book update and trade to connected clients over a **WebSocket**, so the dashboard updates live with zero polling
- Handles concurrent order submission safely — every order acquires a mutex-protected lock on the book before matching

## Architecture

```
┌──────────────────┐    WebSocket / REST    ┌───────────────────────┐    gRPC    ┌───────────────────────┐
│   Next.js UI      │ ─────────────────────▶ │      Go Backend        │ ──────────▶ │  C++ Matching Engine   │
│   (Vercel)         │ ◀───────────────────── │  (Render, Docker)      │ ◀────────── │  (same container)      │
│                    │  live book/trades/stats│  REST API + WS Hub +   │ orders/fills │  price-time priority   │
└──────────────────┘                          │  gRPC client            │             │  order book, matching   │
                                               └───────────────────────┘             └───────────────────────┘
```

- **Go ↔ C++** communicate over **gRPC**, not `cgo` — a clean process boundary that mirrors how real trading systems split a hot-path matching core from the services around it.
- The Go backend and C++ engine ship in **one Docker container** on Render (the engine starts first, backend connects to it over `127.0.0.1:50051`). The Next.js frontend deploys separately to Vercel.

## Tech stack

| Layer | Stack |
|---|---|
| Matching engine | C++17, `std::map` + `std::deque` order book, gRPC server, CMake |
| Backend | Go, `gorilla/websocket`, gRPC client, `net/http` |
| Frontend | Next.js, React, TypeScript, Tailwind CSS |
| Contract | Protocol Buffers (`proto/exchange.proto`) |
| Deployment | Docker (Render), Vercel |

## API surface

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/orders` | Submit a new order (`order_id`, `side`, `price`, `quantity`) |
| `DELETE` | `/orders/{id}` | Cancel a resting order |
| `GET` | `/orderbook` | Current bid/ask book snapshot |
| `GET` | `/trades` | Recent executed trades |
| `GET` | `/stats` | Last price, volume, best bid/ask, spread |
| `GET` | `/health` | Health check (used by Render) |
| `WS` | `/ws` | Live stream of book updates and trades |

Internally, Go talks to the C++ engine over gRPC (`SubmitOrder`, `CancelOrder`, `StreamTrades`, `StreamBookUpdates`) — see `proto/exchange.proto` for the full contract.

## Performance

Load-tested by firing concurrent orders directly at `POST /orders` (`benchmarks/load_test_main.go`), 10,000 orders per concurrency level, randomly priced (mean 100, stddev 2) so a realistic share actually cross and match rather than just stacking up on one side:

| Concurrency | Throughput (req/s) | p50 | p95 | p99 | Errors |
|---:|---:|---:|---:|---:|---:|
| 10 | 7,814.53 | 1.22ms | 1.83ms | 2.29ms | 0.00% |
| 50 | 12,861.43 | 3.66ms | 5.26ms | 6.80ms | 0.00% |
| 100 | 13,562.90 | 7.22ms | 9.36ms | 10.78ms | 0.00% |
| 500 | 13,959.06 | 33.84ms | 46.99ms | 87.28ms | 0.00% |

Throughput plateaus around ~14k req/s while latency keeps climbing under heavier concurrency — the bottleneck is the single global mutex protecting the C++ order book, which forces every order (matching or resting) through a strictly sequential critical section. Full methodology and analysis in [`benchmarks/RESULTS.md`](benchmarks/RESULTS.md).

## Running locally

```bash
git clone https://github.com/Nivadeka222/mini-stock-exchange-simulator.git
cd mini-stock-exchange-simulator
docker compose -f docker/docker-compose.yml up
```

This brings up the C++ engine, Go backend, and Next.js frontend together. The frontend will be on `localhost:3000`, proxying API/WS calls to the backend on `localhost:8080`.

## Repository structure

```
├── engine/       C++ matching engine (order book, gRPC server)
├── backend/      Go REST + WebSocket gateway, gRPC client
├── frontend/     Next.js dashboard
├── proto/        gRPC contract shared by engine and backend
├── benchmarks/   Load test script + results
├── docker/       Dockerfiles (per-service + Render combined build) and local docker-compose
└── docs/         Project write-up
```
