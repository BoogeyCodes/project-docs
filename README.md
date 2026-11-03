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
| **PulseBoard** | Real-time fleet monitoring console with a live dashboard | ASP.NET Core 8, SignalR, React, TypeScript | [docs/pulseboard.md](docs/pulseboard.md) |
| **ShortLink API** | Production-shaped URL shortener | ASP.NET Core 8, MediatR, EF Core | [docs/shortlink-api.md](docs/shortlink-api.md) |
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

- [pulseboard](https://github.com/BoogeyCodes/pulseboard)
- [shortlink-api](https://github.com/BoogeyCodes/shortlink-api)
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
