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
| **ratekit** | Zero-dependency keyed rate limiter (token-bucket + sliding-window) | TypeScript | [docs/ratekit.md](docs/ratekit.md) |
| **logsift** | Zero-dependency log filtering and summarising CLI | Python 3.10+ | [docs/logsift.md](docs/logsift.md) |

## Repositories

- [pulseboard](https://github.com/BoogeyCodes/pulseboard)
- [shortlink-api](https://github.com/BoogeyCodes/shortlink-api)
- [ratekit](https://github.com/BoogeyCodes/ratekit)
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
