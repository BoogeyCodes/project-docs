# cronparse

> Parse cron expressions, compute the next run times, and describe them in plain English.

- **Repository:** https://github.com/BoogeyCodes/cronparse
- **Status:** Complete
- **Primary stack:** Python 3.10+ (standard library only)
- **Last updated:** 2026-11-03

## Overview

cronparse is a small Python library (and CLI) for working with cron expressions. Given a standard
5-field expression, it tells you the exact set of times it matches, computes the upcoming run times
from any starting point, and produces a plain-English description. It has no dependencies, and the
parsing — the part that is easy to get subtly wrong — is the well-tested core.

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
| Dependencies | None | A small, self-contained utility |
| Tests | pytest | Field parsing, matching, next-run and the CLI |
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

Twenty-five tests: field parsing (wildcards, lists, ranges, steps, names, the Sunday `7` alias and
rejected input), matching and `next_run` (the strictly-after rule, weekday schedules, the DOM/DOW OR
semantics, named months, and the impossible-schedule guard), the descriptions, and the CLI. CI runs
on Python 3.10 through 3.13.

## Running and deployment

`pip install -e ".[dev]"` then `pytest`, or install and run `cronparse "<expr>"`. It is a library and
CLI; there is nothing to deploy.

## Challenges and lessons

**The day-field OR rule is the trap.** The non-obvious part of cron is that restricting both day
fields widens the matches rather than narrowing them. Writing the test first (`0 0 1 * MON` must fire
on the 1st and on Mondays) made the rule explicit and kept the implementation honest.

## Future work

- Seconds (6-field) and `@hourly` / `@daily` shorthands.
- Timezone-aware scheduling.
- A previous-run direction.

## Links

- Repository: https://github.com/BoogeyCodes/cronparse
- Related docs: [logsift.md](logsift.md) (another small, scriptable Python tool)
