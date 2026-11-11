# Upcheck

> A self-hosted uptime and incident monitor: scheduled HTTP checks, an up/down state machine with a
> failure threshold, incidents, rolling uptime stats, and a live status page.

- **Repository:** [github.com/BoogeyCodes/upcheck](https://github.com/BoogeyCodes/upcheck)
- **Status:** complete
- **Primary stack:** C# / ASP.NET Core 8, EF Core, hosted BackgroundService
- **Last updated:** 2026-11-11

## Overview

Upcheck watches a set of HTTP endpoints. You register monitors (a URL, an interval, a timeout, an
expected status), and a background scheduler probes each one when it comes due, records the response,
and decides whether the endpoint is up or down. Going down opens an incident; recovering resolves it.
It tracks uptime over rolling windows and serves a self-contained HTML status page alongside a JSON
API. It's the kind of tool a small team self-hosts instead of paying for a status-page service.

## Purpose and goals

The goal was a complete backend whose depth is in *behaviour over time* rather than CRUD: a scheduler,
an HTTP prober, a state machine that resists flapping, incident tracking and uptime maths. A "done"
version lets you add a monitor, have it checked automatically, see it flip down after real failures and
recover, read its uptime, and watch all of it on a status page — with the moving parts (scheduling,
state machine, stats) tested deterministically.

**Non-goals:** multi-user accounts, alerting channels (email/Slack) and distributed checking. The
focus is the monitoring engine; a single shared API key guards writes.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Web | ASP.NET Core 8 Minimal APIs | Thin endpoints over a service layer |
| Scheduling | `BackgroundService` + `PeriodicTimer` | Runs due checks on a tick |
| HTTP | `IHttpClientFactory` | Pooled clients; per-monitor timeout via `CancellationTokenSource` |
| Persistence | EF Core 8 + SQLite | History and incidents with zero infrastructure |
| Errors | RFC 7807 ProblemDetails | Consistent error contract |
| UI | Server-rendered HTML | A dependency-free, auto-refreshing status page |
| Tests | xUnit + FluentAssertions + WebApplicationFactory | Pure unit tests + HTTP integration |

## Architecture

```
SchedulerWorker (hosted)            MonitorRunner.RunDueAsync (scheduled unit of work)
   every 5s ──────────────▶ for each due monitor:
                               IProbe.ProbeAsync  (HTTP GET, per-monitor timeout)
                               → record Check
                               → MonitorEvaluator.Apply  (pure up/down state machine)
                                     ├ WentDown  → open Incident
                                     └ Recovered → resolve Incident
API / status page ─▶ MonitorService ─▶ AppDbContext     (reads: list, detail, stats, status)
```

The worker is a thin loop; all the logic is in `MonitorRunner`, which the tests call directly.
`MonitorEvaluator` and `UptimeCalculator` are pure and have no idea the database or network exist.

## Key design decisions

- **A failure threshold instead of instant flips.** Networks blip. A monitor goes Down only after
  `FailureThreshold` consecutive failures and recovers on the first success. Encoding this in one pure
  method (`MonitorEvaluator.Apply`) makes "does it flap?" a two-line unit test rather than a guess.
- **Everything behind an interface for testing.** `IClock`, `IProbe` and `AppDbContext` are injected,
  so the scheduler's unit of work runs in tests with a fake clock and a *scripted probe* — no real
  sockets, no `Thread.Sleep`, no flaky timing. The integration tests drive a monitor down and back up
  deterministically.
- **Per-monitor timeout.** The probe enforces each monitor's `TimeoutMs` with a linked cancellation
  token and records a timeout as a normal failed check, so a slow endpoint can't stall the scheduler.
- **Sortable timestamps on SQLite.** SQLite can't `ORDER BY` or compare a `DateTimeOffset`, so a value
  converter stores every timestamp as UTC ticks; ordering and windowing stay in SQL.
- **Writes gated, reads open.** An optional `X-Api-Key` protects mutating endpoints while the status
  page and summaries stay public — the split a status page actually wants.

## Data and interface

**Entities:** `Monitor` (url, interval, timeout, expected status, failure threshold, current status,
consecutive failures); `Check` (per-probe outcome: up, status code, response ms, error); `Incident`
(started/resolved span with a cause). Monitors cascade to their checks and incidents.

**REST surface:** `GET/POST /api/monitors`, `GET/PUT/DELETE /api/monitors/{id}`,
`POST /api/monitors/{id}/check` (run now), `GET /api/monitors/{id}/checks` and `/incidents`,
`GET /api/status` (overall + per-monitor), and `GET /` (HTML status page).

## Testing

21 tests. Pure unit tests cover the state machine (one failure doesn't flip; the threshold does;
recovery clears it; a blip doesn't flap), the uptime calculator (percentage, average, windowing) and
the due-time logic. Integration tests boot the real app with a scripted probe and a fake clock and
drive it over HTTP: creating and validating monitors, running a check, taking a monitor **down and
back up** while asserting the incident opens and resolves, the status summary, the HTML status page,
and the API-key gate (401 without the key, 201 with it, reads still public).

## Running and deployment

```bash
dotnet run --project src/Upcheck.Api     # http://localhost:5141  (status page at /, Swagger at /swagger)
dotnet test                              # 21 tests
docker build -t upcheck . && docker run --rm -p 8080:8080 -v upcheck-data:/data upcheck
```

Set `ApiKey` to require a key on writes; set `Scheduler__Enabled=false` for a read-only deployment.
GitHub Actions restores, builds and tests on every push and builds the Docker image.

## Challenges and lessons

- **Naming collision with the BCL.** The core entity is `Monitor`, which clashes with
  `System.Threading.Monitor` under implicit usings. A single global `using` alias
  (`global using Monitor = ...Domain.Monitor;`) fixed it everywhere without renaming the domain.
- **Testing a scheduler without the clock.** Extracting "run the due checks once" into a method that
  takes an injected clock and probe made the whole engine testable synchronously; the `BackgroundService`
  is left as a trivial loop.

## Future work

1. Alerting (email / webhook / Slack) when an incident opens or resolves.
2. Keyword and response-body assertions, not just status codes.
3. Per-monitor response-time charts on the status page.
4. Retention/rollups so the checks table doesn't grow unbounded.
5. Multi-user accounts and per-team status pages.
