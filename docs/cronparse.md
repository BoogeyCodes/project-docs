# cronparse

> Parse, explain and explore cron expressions — in plain English, from the CLI or a web schedule
> explorer.

- **Repository:** https://github.com/BoogeyCodes/cronparse
- **Status:** Complete (v0.2 — added the web explorer)
- **Primary stack:** Python 3.10+ (CLI: standard library only); FastAPI + SQLite for the `[web]` extra
- **Last updated:** 2026-11-25

## Overview

cronparse is a Python library (and CLI) for working with cron expressions. Given a standard 5-field
expression, it tells you the exact set of times it matches, computes the upcoming run times from any
starting point, and produces a plain-English description. The library has no dependencies, and the
parsing — the part that is easy to get subtly wrong — is the well-tested core.

v0.2 adds a **web schedule explorer** (`cronparse serve`, the optional `[web]` extra): type an
expression and see it explained — the description, a field-by-field breakdown, the next and previous
runs with relative times, and a heatmap of runs per hour over the next week — plus a saved-jobs
"crontab" (SQLite) with a combined "what runs next" timeline. Every view has a JSON twin, so the
explorer is also an API, and the compute is shared with the pages.

## Purpose and goals

Everyone has had to decode a line like `*/15 2-6 * * 1-5` by hand, or guess when a scheduled job will
next fire. The goal was a small tool that answers both precisely: parse the expression into the values
it allows, compute the next runs, and describe the schedule in words — with the awkward corners of
cron (steps, name aliases, the day-of-month/day-of-week rule) handled correctly.

Non-goals: 6-field (seconds) cron, `@hourly`-style shorthands, and timezone handling are listed as
future work; the library is timezone-naive.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | Python 3.10+ | Sets and `datetime` make the logic direct |
| CLI dependencies | None | A small, self-contained utility |
| Explorer (`[web]` extra) | FastAPI + Uvicorn + Jinja2 | A schedule explorer and JSON API over the parser |
| Persistence | `sqlite3` (standard library) | Saved named jobs for the combined timeline |
| Tests | pytest | Field parsing, matching, next-run, the CLI, and the explorer |
| CI | GitHub Actions matrix | Python 3.10–3.13 |

## Architecture

```
  "*/15 9-17 * * 1-5"
        │ split into 5 fields
        ▼
  parse_field(raw, low, high, names)   each field -> frozenset of allowed ints
        │
        ▼
  CronExpression(minute, hour, dom, month, dow)
    matches(dt)    set-membership per field + the DOM/DOW OR rule
    next_run(after) step forward minute by minute until a match (capped)
    describe()     plain-English summary
        │
        ▼
  cli.py   describe + list the next N runs
```

The explorer reuses that engine rather than reimplementing it. `schedule.py` holds pure helpers over a
`CronExpression` — the field breakdown, the upcoming-runs list with relative times, and a run-density
heatmap built by scanning the next week minute by minute (bounded regardless of how sparse the
schedule is, unlike repeatedly calling `next_run`). `web.py` turns those into HTML and the same shape
of JSON, and a small SQLite `JobStore` keeps saved jobs so the explorer can show a merged timeline.

## Key design decisions

**Each field becomes the set of values it allows.** `*/15` on the minute field expands to
`{0, 15, 30, 45}`, `1-5` to `{1,2,3,4,5}`, `MON` to `{1}`. Matching a time is then a membership check
per field, which is simple and obviously correct. Parsing handles wildcards, lists, ranges, steps
(including `a/step` meaning "from a to the maximum") and month/weekday names, with `7` aliased to
Sunday.

**The day-of-month OR day-of-week rule is implemented faithfully.** Standard cron has a quirk: when
*both* the day-of-month and day-of-week fields are restricted, a day matches if *either* matches
(not both). cronparse detects which fields are wildcards and applies exactly that rule, with a test
pinning the behaviour (`0 0 1 * MON` fires on the 1st *and* on every Monday).

**`next_run` steps forward, but with a cap.** Finding the next run walks forward minute by minute
until a match. That is simple and correct, and a five-year cap turns an impossible schedule (the 30th
of February) into a clear error instead of an infinite loop.

**Descriptions are derived from the parsed sets.** The English summary detects common shapes — a
single time (`At 09:00`), an even step (`Every 15 minutes`), named weekdays and months — from the
value sets rather than re-parsing the raw text.

## Data and interface

`CronExpression.parse(str)` yields an object with `matches(dt)`, `next_run(after)`,
`next_runs(count, after)`, `iter_runs(after)` and `describe()`. Parsing errors raise `CronError`
(a `ValueError`). The CLI is `cronparse "<expr>" [-n COUNT]`.

## Testing

Forty tests: field parsing (wildcards, lists, ranges, steps, names, the Sunday `7` alias and rejected
input), matching and `next_run` (the strictly-after rule, weekday schedules, the DOM/DOW OR semantics,
named months, and the impossible-schedule guard), the descriptions, and the CLI. The explorer adds
tests for the schedule helpers (field views, upcoming runs, relative times, and a heatmap that stays
bounded even for a yearly schedule), the job store, and the FastAPI app via `TestClient` (explaining
valid and invalid expressions, the JSON API, and saving/deleting jobs). CI runs on Python 3.10–3.13.

## Running and deployment

`pip install -e ".[dev]"` then `pytest`, or install and run `cronparse "<expr>"`. The explorer is
`pip install "cronparse[web]"` then `cronparse serve`, or the included Docker image (saved jobs
persist in a mounted `/data` volume).

## Challenges and lessons

**The day-field OR rule is the trap.** The non-obvious part of cron is that restricting both day
fields widens the matches rather than narrowing them. Writing the test first (`0 0 1 * MON` must fire
on the 1st and on Mondays) made the rule explicit and kept the implementation honest.

## Future work

- Seconds (6-field) and `@hourly` / `@daily` shorthands.
- Timezone-aware scheduling.
- Comparing two expressions side by side in the explorer.

## Links

- Repository: https://github.com/BoogeyCodes/cronparse
- Related docs: [logsift.md](logsift.md) (another small, scriptable Python tool)
