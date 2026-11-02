# PulseBoard

> A real-time equipment monitoring console: devices stream telemetry, rules turn it into alerts, and
> operators watch and act on a live dashboard.

- **Repository:** https://github.com/BoogeyCodes/pulseboard
- **Status:** Complete, actively maintained
- **Primary stack:** ASP.NET Core 8, SignalR, EF Core + Dapper, React 19, TypeScript, Redux Toolkit
- **Last updated:** 2026-11-02

## Overview

PulseBoard is a full-stack monitoring system for a fleet of devices. Each device reports CPU,
temperature and latency on a short interval. The backend ingests those samples, stores them,
evaluates threshold rules, maintains each device's current status, raises and resolves alerts, and
pushes every change to connected browsers over a WebSocket. The frontend is a single-screen dark
dashboard: summary tiles, a device grid, live per-device charts and an alert panel where operators
acknowledge and resolve issues.

It is built as a reference for how I structure a production-shaped full-stack system end to end:
authentication and authorization, a background processing pipeline, real-time transport, two data
access styles over one database, a typed React state layer, tests on both sides, and a single Docker
image that ships the whole thing. It mirrors the kind of operational software I build professionally,
compressed into something a reviewer can read in an afternoon.

## Purpose and goals

The problem: operational teams need to see the state of many devices at once, be told immediately
when something goes wrong, and record that a human has responded. Polling a REST API every few
seconds does not scale and feels dead; the data needs to arrive the moment it changes.

Goals:

- Ingest telemetry without the producer ever blocking, and without unbounded memory growth under load.
- Turn raw samples into a small, meaningful set of alerts, with no duplicate noise when a condition
  persists, and automatic clearing when it passes.
- Push every change to every connected dashboard in real time, and recover cleanly after a dropped
  connection.
- Enforce who can do what on the server, not just in the UI.
- Be runnable by anyone with Docker in one command, with live demo data from the first second.

Non-goals: this is a single-instance demo. Horizontal scaling (a shared database and a SignalR
backplane) is designed for but not implemented, and the telemetry source is a simulator rather than
real hardware.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| API framework | ASP.NET Core 8 Minimal APIs | Low-ceremony endpoints with typed results and first-class OpenAPI |
| Ingest pipeline | `System.Threading.Channels` | Bounded, in-process producer/consumer queue with explicit back-pressure |
| Background work | `BackgroundService` hosted services | The channel consumer, the simulator and the heartbeat sweep |
| Real-time | SignalR with a typed `Hub<ITelemetryClient>` | Server cannot misspell a client method; reconnection handled by the client library |
| Persistence (write) | Entity Framework Core 8 (SQLite) | Owns the schema and all writes; provider swaps to PostgreSQL in one line |
| Persistence (read) | Dapper | The hot time-series read and the retention purge, with explicit SQL and index usage |
| Auth | JWT bearer, `PasswordHasher<T>`, policy-based authorization | Stateless tokens that also authenticate the WebSocket; capability policies over role strings |
| Logging | Serilog | Structured request logging and searchable properties |
| Frontend | React 19 + TypeScript | Typed components and hooks |
| State | Redux Toolkit | One store fed by a background WebSocket and shared across every view |
| Charts | Recharts | Declarative live line charts |
| Build/ship | Vite, multi-stage Dockerfile, GitHub Actions | One image serves API, hub and SPA; CI tests both sides and smoke-tests the container |

## Architecture

```
  simulator / real ingest
         │ writes
         ▼
  Channel<TelemetrySample>  (bounded 1024, drop-oldest, single reader)
         │ reads
         ▼
  TelemetryWorker (one consumer)
         │  per sample, in a scoped TelemetryProcessor:
         │    • persist Reading                     (EF Core)
         │    • update Device last-known state      (EF Core)
         │    • evaluate rules -> raise / escalate / auto-resolve Alerts
         │    • broadcast changes                   (IHubContext)
         └─ every 5s: mark silent devices offline, purge old readings (Dapper)
         │
         ▼ broadcasts over
  SignalR hub  /hubs/telemetry
         │ DeviceUpdated · ReadingReceived · AlertRaised · AlertUpdated
         ▼
  React app
    useTelemetryHub  -> dispatches into Redux store
    slices: auth · devices · alerts · readings (ring buffer) · connection
    views: SummaryBar · DeviceGrid · DeviceDetail (charts) · AlertsPanel
```

The core idea is a single writer. Many producers can push samples into the channel, but exactly one
consumer touches device and alert state, so there are no races and no locks. The REST API is a
read-and-command surface on the same database; it never processes telemetry itself.

## Key design decisions

**A bounded channel with drop-oldest, not an unbounded queue.** Telemetry is a firehose. An
unbounded queue trades a crash for a slow memory leak under load. A bounded channel that drops the
oldest sample when full keeps memory flat and loses the least useful data first (the stalest
reading), which is the right failure mode for monitoring.

**Rules are pure; reconciliation is stateful.** `AlertRules.Evaluate(sample)` is a pure function
returning the conditions that currently hold. The processor compares that to the open alerts and
decides what to raise, escalate, de-escalate or auto-resolve. Separating the two makes the rules
trivial to unit test and keeps the stateful database logic in one place.

**EF Core and Dapper together.** EF Core owns the model and writes, where its change tracking and
migrations earn their keep. The one hot path — reading a device's recent time series — and the
retention purge use Dapper, so the SQL and its index usage are explicit and fast. Using the right
tool for each job is itself the point.

**The JWT authenticates the WebSocket.** Browsers cannot set headers on a WebSocket upgrade, so the
token rides on the query string and a `JwtBearerEvents` hook reads it only for the hub path. One auth
scheme covers both REST and real-time.

**Authorization is capability-based.** Endpoints require `CanView`, `CanOperate` or `CanAdminister`,
not raw role names. The mapping from capability to roles lives in one place, so changing who can
operate is a one-line edit, and the endpoints read like documentation.

## Data and interface

**Entities.** `Device` (id, name, site, status, last-known metrics), `Reading` (device, timestamp,
cpu, temp, latency), `Alert` (device, kind, severity, raised/acknowledged/resolved timestamps and
actors), `User` (username, password hash, role).

**REST surface.**

```
POST /api/auth/login                 username+password -> JWT
GET  /api/auth/me                     current token's identity
GET  /api/devices                     all devices with last-known state
GET  /api/devices/{id}                one device
GET  /api/devices/{id}/readings       recent telemetry, oldest first
GET  /api/alerts                      alerts (open by default), newest first
POST /api/alerts/{id}/acknowledge     operator only
POST /api/alerts/{id}/resolve         operator only
GET  /api/summary                     fleet counts for the tiles
```

**Hub (server to client):** `DeviceUpdated`, `ReadingReceived`, `AlertRaised`, `AlertUpdated`.

## Testing

Twenty-three backend tests boot the real application with `WebApplicationFactory`, the simulator
switched off, a throwaway SQLite file and a fake clock. Pipeline tests push samples through the
processor exactly as the worker would, then assert over the REST API and over a real SignalR client
connection. Expiry and offline detection are tested by advancing the fake clock, never by sleeping,
so the suite is fast and deterministic.

Eight frontend tests cover the Redux reducers and selectors against a real store, the components with
Testing Library, and the acknowledge flow against a stubbed `fetch` that asserts the bearer header is
sent. CI runs both suites plus lint, type-check and a Docker build that smoke-tests the running
container.

## Running and deployment

`docker compose up --build` builds the SPA, builds and tests the API, and serves everything from one
container on port 8080. Sign in as `operator` / `pulse123`. For development, the API runs on 5100 and
the Vite dev server on 5173 proxying to it. One multi-stage Dockerfile does frontend build, backend
test and publish, and a slim runtime stage that serves the API, the hub and the built React app
together.

## Challenges and lessons

**SQLite and `DateTimeOffset`.** SQLite stores `DateTimeOffset` as text and cannot order or compare
it in SQL. This surfaced twice: the alert list query threw at runtime, and the Dapper time-series
query compared timestamps as strings. The fix was to order alerts by their monotonic id instead, and
to format Dapper's parameters in the exact text layout EF Core uses so lexicographic order matches
chronological order. The lesson: know how your provider actually stores each type.

**The container exited on first run in CI.** The .NET runtime image runs as a non-root `app` user,
which could not create the SQLite file in the root-owned data volume. CI caught it because the Docker
job smoke-tests the running container, not just the build. The fix was one `chown` line. Worth having
CI run the thing, not only compile it.

## Future work

- PostgreSQL with EF migrations and a Redis backplane for SignalR, so several API instances share load.
- A real ingest producer (MQTT or HTTP) writing into the same channel.
- Alert routing to email or Slack with per-site on-call rules.
- Admin-editable per-device thresholds instead of constants.
- OpenTelemetry traces and metrics; the structured logs are already in place.

## Links

- Repository: https://github.com/BoogeyCodes/pulseboard
- Related docs: [shortlink-api.md](shortlink-api.md) (the same backend patterns at smaller scale)
