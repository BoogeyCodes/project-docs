# Project Documentation

A single place for the design docs behind my projects. Code lives in each project's own
repository; this repo holds the **why** and the **how** — purpose, stack, architecture, the
decisions I made and what I'd do differently. Every project gets at least a two-page write-up.

I keep docs separate from code on purpose. A README sells the project to a visitor in thirty
seconds; these documents are the longer engineering story, the kind a reviewer or a client reads
when they want to know how I think.

## Index

| Project | One line | Primary stack | Doc |
| --- | --- | --- | --- |
| **Gambit** | A playable chess engine: perft-verified move generation, alpha-beta search, browser board | TypeScript, Vite | [docs/gambit.md](docs/gambit.md) |
| **Upcheck** | Self-hosted uptime monitor: scheduled checks, incidents, uptime stats, status page | C# / ASP.NET Core 8, EF Core | [docs/upcheck.md](docs/upcheck.md) |
| **FairShare** | Shared-expense tracker: groups, flexible splits, balances and minimal settle-up | Python, FastAPI, SQLAlchemy, HTMX | [docs/fairshare.md](docs/fairshare.md) |
| **TaskForge** | Team project & issue-tracker API with real-time board updates | C# / ASP.NET Core 8, EF Core, SignalR | [docs/taskforge.md](docs/taskforge.md) |
| **AgentScore** | Web app to rate, rank and export reviews of AI coding-agent runs | React, TypeScript | [docs/agentscore.md](docs/agentscore.md) |
| **agentrubric** | Agent-evaluation platform: score/rank runs, a dashboard, REST API and a cross-evaluation leaderboard | Python, FastAPI | [docs/agentrubric.md](docs/agentrubric.md) |
| **runtrace** | Normalise agent run transcripts into one schema + a browser **run-comparison dashboard** (metrics, bars, shared files) | TypeScript, Vite | [docs/runtrace.md](docs/runtrace.md) |
| **patchkit** | Parse, summarise and apply the unified diffs agents produce | Python | [docs/patchkit.md](docs/patchkit.md) |
| **sandrun** | Sandbox-execution service: run agent work against tests (local/Docker), a dashboard, REST API and pass-rate tracking | Python, FastAPI | [docs/sandrun.md](docs/sandrun.md) |
| **difflens** | Side-by-side multi-run diff views | TypeScript | [docs/difflens.md](docs/difflens.md) |
| **Trackpense** | Personal finance tracker: transactions, budgets, live dashboard | C# / Blazor Server, EF Core | [docs/trackpense.md](docs/trackpense.md) |
| **PulseBoard** | Real-time fleet monitoring console with a live dashboard | ASP.NET Core 8, SignalR, React, TypeScript | [docs/pulseboard.md](docs/pulseboard.md) |
| **ShortLink API** | Production-shaped URL shortener | ASP.NET Core 8, MediatR, EF Core | [docs/shortlink-api.md](docs/shortlink-api.md) |
| **statemachine** | Declarative finite state machine with guards and callbacks | Python | [docs/statemachine.md](docs/statemachine.md) |
| **expreval** | Safe arithmetic expression evaluator (shunting-yard), no `eval` | Python | [docs/expreval.md](docs/expreval.md) |
| **globlite** | Glob matcher (`*`, `**`, `?`, classes, braces) compiled to RegExp | TypeScript | [docs/globlite.md](docs/globlite.md) |
| **tinycsv** | RFC 4180 CSV parser and stringifier | TypeScript | [docs/tinycsv.md](docs/tinycsv.md) |
| **Outcome** | Result and Option functional types | C# / .NET 8 | [docs/outcome.md](docs/outcome.md) |
| **tabulate** | Render CSV/TSV as aligned ASCII or Markdown tables | C# / .NET 8 | [docs/tabulate.md](docs/tabulate.md) |
| **cronparse** | Parse cron expressions, compute next runs, describe in English | Python | [docs/cronparse.md](docs/cronparse.md) |
| **fuzzykit** | String similarity and fuzzy search (Levenshtein, Jaro-Winkler, Dice) | TypeScript | [docs/fuzzykit.md](docs/fuzzykit.md) |
| **bytefmt** | `DataSize` value type + **DiskScope**, a disk-usage explorer (scan engine, dashboard, REST API) | C# / .NET 8, ASP.NET Core | [docs/bytefmt.md](docs/bytefmt.md) |
| **quickpoll** | Polls API: create polls, vote once per voter, live results | Ruby on Rails | [docs/quickpoll.md](docs/quickpoll.md) |
| **recall** | Spaced-repetition flashcards with an SM-2 scheduler | Angular | [docs/recall.md](docs/recall.md) |
| **contrastcheck** | WCAG colour-contrast checker | React | [docs/contrastcheck.md](docs/contrastcheck.md) |
| **gridcalc** | Spreadsheet formula engine: parser, dependency graph, cycle detection | TypeScript | [docs/gridcalc.md](docs/gridcalc.md) |
| **sketchkit** | Probabilistic data structures: Bloom, Count-Min, HyperLogLog | Python | [docs/sketchkit.md](docs/sketchkit.md) |
| **ratekit** | Zero-dependency keyed rate limiter (token-bucket + sliding-window) | TypeScript | [docs/ratekit.md](docs/ratekit.md) |
| **flagpole** | Feature-flag service with deterministic percentage rollouts | Python, FastAPI | [docs/flagpole.md](docs/flagpole.md) |
| **jsondiff** | CLI that structurally diffs two JSON files | C# / .NET 8 | [docs/jsondiff.md](docs/jsondiff.md) |
| **logsift** | Zero-dependency log filtering and summarising CLI | Python 3.10+ | [docs/logsift.md](docs/logsift.md) |

## Repositories

- [gambit](https://github.com/BoogeyCodes/gambit)
- [upcheck](https://github.com/BoogeyCodes/upcheck)
- [fairshare](https://github.com/BoogeyCodes/fairshare)
- [taskforge](https://github.com/BoogeyCodes/taskforge)
- [agentscore](https://github.com/BoogeyCodes/agentscore)
- [agentrubric](https://github.com/BoogeyCodes/agentrubric)
- [runtrace](https://github.com/BoogeyCodes/runtrace)
- [patchkit](https://github.com/BoogeyCodes/patchkit)
- [sandrun](https://github.com/BoogeyCodes/sandrun)
- [difflens](https://github.com/BoogeyCodes/difflens)
- [trackpense](https://github.com/BoogeyCodes/trackpense)
- [pulseboard](https://github.com/BoogeyCodes/pulseboard)
- [shortlink-api](https://github.com/BoogeyCodes/shortlink-api)
- [statemachine](https://github.com/BoogeyCodes/statemachine)
- [expreval](https://github.com/BoogeyCodes/expreval)
- [globlite](https://github.com/BoogeyCodes/globlite)
- [tinycsv](https://github.com/BoogeyCodes/tinycsv)
- [outcome](https://github.com/BoogeyCodes/outcome)
- [tabulate](https://github.com/BoogeyCodes/tabulate)
- [cronparse](https://github.com/BoogeyCodes/cronparse)
- [fuzzykit](https://github.com/BoogeyCodes/fuzzykit)
- [bytefmt](https://github.com/BoogeyCodes/bytefmt)
- [quickpoll](https://github.com/BoogeyCodes/quickpoll)
- [recall](https://github.com/BoogeyCodes/recall)
- [contrastcheck](https://github.com/BoogeyCodes/contrastcheck)
- [gridcalc](https://github.com/BoogeyCodes/gridcalc)
- [sketchkit](https://github.com/BoogeyCodes/sketchkit)
- [ratekit](https://github.com/BoogeyCodes/ratekit)
- [flagpole](https://github.com/BoogeyCodes/flagpole)
- [jsondiff](https://github.com/BoogeyCodes/jsondiff)
- [logsift](https://github.com/BoogeyCodes/logsift)

## How these docs are organised

Each document follows [TEMPLATE.md](TEMPLATE.md) and covers, at minimum:

1. **Overview** — what it is and who it's for, in a paragraph.
2. **Purpose and goals** — the problem it solves and what "done" means.
3. **Tech stack** — every significant technology and why it's there.
4. **Architecture** — how the pieces fit, with a diagram.
5. **Key design decisions** — the choices worth explaining, and the trade-offs.
6. **Data and interface** — the model, the API or CLI surface.
7. **Testing** — what is tested and how.
8. **Running and deployment** — how to build, run and ship it.
9. **Challenges and lessons** — what was hard and what I learned.
10. **Future work** — where it would go next.

## Adding a new project

1. Copy [TEMPLATE.md](TEMPLATE.md) to `docs/<project>.md`.
2. Fill every section. Aim for two pages or more; a half-page stub helps no one.
3. Add a row to the index table above.

## Author

Abdul Raheem · [github.com/BoogeyCodes](https://github.com/BoogeyCodes) · abdulraheem07170@gmail.com
