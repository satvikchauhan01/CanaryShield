# CanaryShield

**Feature flags and canary releases with automatic rollback and blast-radius control.**

CanaryShield releases a new feature to a small, controlled slice of users first, watches that slice's error rate live in Prometheus, and **rolls the feature back automatically** the moment it starts failing. A bad release only ever reaches a few users, never everyone, and nobody has to be awake to fix it.

> Hackathon project by **Satvik** (backend) and **Krish** (frontend).

![Release overview](docs/screenshots/dashboard-overview.png)

## Why

A normal deploy sends a new version to 100% of users at once. If it has a bug, everyone is hit before anyone notices, and rolling back needs a person to find the problem and redeploy. CanaryShield turns that into a controlled experiment:

1. Serve the new version to **5%** of users, then **10% → 25% → 50% → 100%**.
2. Compare the new version's (the *canary's*) error rate with the stable version's (the *baseline*).
3. If the canary stays above its threshold for consecutive checks, the **guardian** rolls it back to 0% and opens an **incident** that records who was exposed and how fast it was stopped.

## Features

- **Feature flag management:** draft → rolling out → paused → completed / rolled back, plus an instant kill switch.
- **Targeting:** conditions on any user attribute (`eq`, `neq`, `in`, `not_in`, `gt`, `lt`, `exists`) and per-user include/exclude overrides.
- **Deterministic rollout:** users are bucketed with `sha256(flagKey:salt:userId) mod 10000`, so the same user always gets the same answer and raising the percentage only ever *adds* users.
- **Guardian:** checks Prometheus every 5 seconds; rolls back after consecutive breaches, never on missing data.
- **Rollout control center:** every risky action shows *current state → new state → expected impact* before you confirm.
- **Live dashboard:** health score, canary-vs-baseline charts (error rate, p95 latency, traffic), incidents with timelines, and a full audit log, all updated over Server-Sent Events.
- **Playground:** ask "what would this user get, and why?" or simulate hundreds of users, without affecting real traffic.
- **Chaos testing:** inject failures into the demo app's new payment flow, with a live "what will happen" preview and an emergency stop.
- **QuickCart demo shop:** a food-ordering app whose checkout is released through CanaryShield.

| Rollout control center | Incident detail |
|---|---|
| ![Rollout control center](docs/screenshots/rollout-control.png) | ![Incident detail](docs/screenshots/incident-detail.png) |

| Chaos testing | Playground |
|---|---|
| ![Chaos testing](docs/screenshots/chaos-testing.png) | ![Playground](docs/screenshots/playground.png) |

## Architecture

```mermaid
flowchart LR
    Dash["Dashboard<br/>React · :5173"] -- "REST + SSE" --> API
    Shop["QuickCart web<br/>React · :5174"] --> QCS
    QCS["QuickCart server<br/>Node/Express · :4000"] -- "POST /sdk/v1/evaluate<br/>old or new?" --> API
    API["Platform API<br/>Go · :8080<br/>flags · rollout · evaluation · guardian"]
    API --> PG[("PostgreSQL<br/>source of truth")]
    API --> RD[("Redis<br/>cache · pub/sub · counters")]
    QCS -- "/metrics" --> PROM[("Prometheus<br/>:9090")]
    API -. "guardian queries<br/>every 5 s" .-> PROM
    LG["Load generator<br/>Go"] -- "POST /api/pay" --> QCS
```

| Component | Role |
|---|---|
| **Platform API** (`platform/`) | Owns flags, rollout state, targeting and evaluation. Runs the guardian and the live event stream. |
| **PostgreSQL** | Source of truth: flags, rules, rollout events, incidents, audit log. Every change is one transaction. |
| **Redis** | Compiled flag cache, change notifications (pub/sub), guardian health records, unique-user exposure counts (HyperLogLog). |
| **Prometheus** | Scrapes QuickCart's payment metrics every 5 s; queried by the guardian and the dashboard charts. |
| **Dashboard** (`dashboard/`) | The operator UI. |
| **QuickCart** (`quickcart/`) | The demo app: a Node/Express server (payments, chaos, `/metrics`) and a React storefront. |
| **Load generator** (`loadgen/`) | Simulated shoppers so the guardian always has traffic to judge. |

A deeper explanation of the data flows and design decisions is in [`docs/architecture.md`](docs/architecture.md).

## Tech stack

| Layer | Technology |
|---|---|
| Platform backend | Go 1.25, chi, pgx, go-redis, Prometheus client |
| Database / cache | PostgreSQL 16, Redis 7 |
| Monitoring | Prometheus |
| Dashboard and QuickCart web | React 18, TypeScript, Vite, Tailwind CSS, TanStack Query, react-router, Recharts, lucide-react |
| QuickCart server | Node.js 20+, Express, prom-client |
| Infrastructure | Docker Compose |

## Getting started

### Prerequisites

Docker Desktop, Go 1.25+, Node.js 20+, and Git Bash (on Windows) or any POSIX shell.

### Quick start (Windows)

From the repository root, start everything with one command:

```powershell
.\dev-start.ps1
```

This starts PostgreSQL, Redis and Prometheus with Docker, then opens one window each for the platform (`:8080`), the QuickCart server (`:4000`), the dashboard (`:5173`) and the QuickCart shop (`:5174`). Install dependencies once first with `npm install` in `dashboard/`, `quickcart/server/` and `quickcart/web/`.

`dev-start.ps1` expects Go at `C:\Program Files\Go\bin\go.exe`; edit the `$go` variable at the top if yours is elsewhere.

### Manual start

Run each in its own terminal, in this order:

```bash
# 1. Infrastructure
docker compose up -d

# 2. Platform API (:8080). Migrations are applied by Postgres on first start.
cd platform && go run ./cmd/server

# 3. QuickCart server (:4000)
cd quickcart/server && npm install && npm run dev

# 4. Dashboard (:5173)
cd dashboard && npm install && npm run dev

# 5. QuickCart shop (:5174)
cd quickcart/web && npm install && npm run dev
```

Then seed the demo flag and start some traffic:

```bash
# Creates new_payment_flow at 0% with the default guardrail
bash scripts/seed.sh

# Simulated shoppers: 30 orders per second for 30 minutes
cd loadgen && go run . -rps 30 -duration 30m
```

Open the dashboard at <http://localhost:5173> and the shop at <http://localhost:5174>.

### Configuration

Every app runs with sensible local defaults and no `.env` file. To override anything, copy [`.env.example`](.env.example) to `.env`.

| Variable | Default | Used by |
|---|---|---|
| `DATABASE_URL` | `postgres://flagguard:flagguard@localhost:5432/flagguard?sslmode=disable` | Platform |
| `REDIS_URL` | `redis://localhost:6379/0` | Platform |
| `PROMETHEUS_URL` | `http://localhost:9090` | Platform |
| `PORT` | `8080` | Platform |
| `ADMIN_TOKEN` | `dev-admin-token` | Platform (admin API), Dashboard (`VITE_ADMIN_TOKEN`) |
| `SDK_API_KEY` | `dev-sdk-key` | Platform, QuickCart server |
| `CORS_ORIGINS` | `http://localhost:5173,http://localhost:5174` | Platform, QuickCart server |
| `PLATFORM_URL` | `http://localhost:8080` | QuickCart server |
| `VITE_PLATFORM_URL` / `VITE_QUICKCART_URL` | `:8080` / `:4000` | Dashboard, QuickCart web |
| `VITE_USE_MOCKS` | `false` | Dashboard: `true` runs the UI on built-in sample data without a backend |

> The default tokens are for local development only. Set your own `ADMIN_TOKEN` and `SDK_API_KEY` before exposing the platform to a network.

## The demo, step by step

1. **Dashboard → Feature flags → New payment flow → Rollout tab.** The flag is at 0%: everyone uses the stable payment flow.
2. Click **Start rollout at 5%**, confirm, then **Advance** to 10% and 25%. Each step shows a confirmation with the expected impact.
3. In **QuickCart** (`:5174`), pick *Demo Canary* under the persona menu, add a dish and pay. The confirmation shows **New payment flow · canary**.
4. **Dashboard → Chaos testing → Demo failure → Start experiment.** The new payment flow now fails 40% of the time.
5. Watch the **Health** tab: the canary's error rate climbs past the 3% threshold and the status goes *Warning → Breached*.
6. Within seconds the **guardian rolls the flag back to 0%** with no one touching anything. A toast and a red banner appear, and an incident is opened.
7. Pay again in QuickCart: you are back on the **Classic payment flow**.
8. Open the **incident**: how much traffic was exposed, how many users, and the time to rollback.

Stop chaos, resolve the incident and run `bash scripts/seed.sh` to reset before the next run.

## Project structure

```text
.
├── platform/            Go platform API: flags, rollout, targeting, evaluation, guardian, SSE
│   ├── cmd/server/      Entry point
│   └── internal/        httpapi · flags · rollout · targeting · evaluation · cache
│                        monitoring · guardian · audit · stream · config · db · models
├── dashboard/           React + TypeScript operator dashboard
├── quickcart/
│   ├── server/          Node/Express demo app: payments, chaos, personas, /metrics
│   └── web/             React storefront
├── loadgen/             Go load generator (simulated shoppers)
├── infra/               SQL migrations and Prometheus config
├── scripts/seed.sh      Seeds the demo flag
├── docs/                Architecture notes, demo script, screenshots
├── client/              Early standalone prototype (not used by the system)
├── docker-compose.yml   PostgreSQL, Redis, Prometheus
└── dev-start.ps1        One-command local start (Windows)
```

## API overview

Admin endpoints live under `/api/v1` and need `Authorization: Bearer <ADMIN_TOKEN>`. Errors use one format: `{"error":{"code":"…","message":"…"}}`.

| Method | Path | Purpose |
|---|---|---|
| `GET` `POST` | `/api/v1/flags` | List / create flags |
| `GET` `PATCH` | `/api/v1/flags/{key}` | Read / edit a flag |
| `PUT` | `/api/v1/flags/{key}/conditions` · `/overrides` · `/guardrail` | Targeting, overrides, guardrail |
| `POST` | `/api/v1/flags/{key}/rollout/{start\|advance\|set\|pause\|resume}` | Rollout control |
| `POST` | `/api/v1/flags/{key}/rollback` · `/kill` | Roll back (with reason) / kill switch |
| `GET` | `/api/v1/flags/{key}/health` · `/metrics` · `/events` | Guardian reading, chart series (`?range=5m…24h`), activity |
| `GET` `POST` | `/api/v1/incidents` · `/{id}/resolve` | Automatic-rollback incidents |
| `POST` | `/api/v1/playground/evaluate` | Side-effect-free evaluation |
| `GET` | `/api/v1/stream?token=` | Live updates (Server-Sent Events) |
| `POST` | `/sdk/v1/evaluate` | What client apps call per request (`X-API-Key` header) |

## Development

```bash
# Platform
cd platform && go build ./... && go vet ./... && go test ./...

# Dashboard and QuickCart web
cd dashboard && npm run lint && npm run build
cd quickcart/web && npm run lint && npm run build
```

The pure logic (bucketing, the rollout state machine, targeting operators and evaluation order) is unit-tested in Go.

## Design notes

- **Fail safe to stable.** Every uncertain case serves the old version: unknown flag, draft, killed or rolled-back flag, failed targeting, no user ID, or a platform that does not answer within 50 ms.
- **The guardian never acts on missing data.** If Prometheus is down or returns nothing, it skips the tick instead of rolling back.
- **No stale overwrites.** The guardian's rollback only succeeds if the flag is still at the version it checked, so it can never undo a newer human decision.
- **PostgreSQL is the source of truth, Redis is only a cache.** Everything in Redis can be rebuilt.

## Limitations

- One environment (`Local`); staging/production and A/B experiments are shown in the UI as *Planned*.
- A single shared admin token; there are no user accounts or roles yet.
- The guardian decides on error rate only. Latency is charted but not enforced.
- Owner and tags are not stored on flags.

## Team

| | Role | Owns |
|---|---|---|
| **Satvik** | Backend | Platform API, load generator, infrastructure, scripts |
| **Krish** | Frontend | Dashboard, QuickCart shop and server |
