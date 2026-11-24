# logsift

> Filter and summarise JSON or plain-text application logs — from the command line or a web console.

- **Repository:** https://github.com/BoogeyCodes/logsift
- **Status:** Complete (v0.2 — added the web console)
- **Primary stack:** Python 3.10+ (CLI: standard library only); FastAPI + SQLite for the `[web]` extra
- **Last updated:** 2026-11-24

## Overview

logsift reads log files — or standard input — understands both structured JSON-lines output and
classic `timestamp LEVEL message` text, and lets you ask questions of them. Two CLI modes: `filter`
prints the matching lines, and `stats` summarises them. It answers "what errored in the last fifteen
minutes?" or "which messages dominate this file?" without a `jq` incantation, and it streams, so
multi-gigabyte files are fine.

It exists for the gap between a raw log file and a log platform. A lot of debugging starts with a
`kubectl logs` dump or a `.log.gz` pulled off a server — files that never reached a search index.
logsift is the small tool for exactly that moment.

v0.2 adds a **web console** (`logsift serve`, the optional `[web]` extra) over the same engine: upload
or paste logs and get a dashboard — the level breakdown, a timeline of volume with errors picked out
in red, the message patterns behind the noise, and the top errors — plus a viewer that filters by
level, text, structured field and time window, live. Every view has a JSON twin, so the console is
also an API. The CLI itself stays dependency-free.

## Purpose and goals

Goals: understand the many shapes of real-world logs without configuration; normalise levels and
timestamps across ecosystems so filtering is uniform; never silently drop a line it cannot parse;
stream rather than load everything into memory; and behave well in a pipe, including sensible exit
codes for scripting. "Done" means it handles Serilog, pino, Python logging and syslog shapes out of
the box and is proven against real line formats in tests.

Non-goals: it is not a log shipper, a tail-follow tool or a replacement for a search platform. It is a
read-only analysis tool for files and streams you already have.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | Python 3.10+ | Ubiquitous on servers and developer machines |
| CLI dependencies | None (standard library only) | Installs and runs anywhere Python does; nothing to audit |
| CLI | `argparse` with subcommands | Familiar interface, built in |
| Console (`[web]` extra) | FastAPI + Uvicorn + Jinja2 | A dashboard and JSON API over the same engine |
| Persistence | `sqlite3` (standard library) | Uploaded datasets parsed once and stored |
| Packaging | `pyproject.toml`, console entry point | `pip install` gives a `logsift` command |
| Lint | ruff | Fast, strict, zero-config |
| Testing | pytest | Proves the parsers against real shapes; also the store and console |
| CI | GitHub Actions matrix | Linux and Windows, Python 3.10 to 3.13 |

## Architecture

```
  files / stdin / .gz
        │
        ▼
  parsers.py   JSON-lines first, text fallback; never fails a line
        │ Record(timestamp, level, message, fields, source, raw)
        ▼
  filters.py   FilterSpec -> one predicate (level, time window, grep, exclude, where)
        │ matching records
        ▼
  ┌─────────────┴──────────────┐
  ▼                            ▼
  cli.py: filter               stats.py: summarise
  pretty / raw / json output   templating + time buckets -> text or JSON
```

Everything is a lazy generator from end to end, so memory stays flat regardless of file size. The
`Record` dataclass is the one shape that flows through parsing, filtering and summarising.

The console reuses that pipeline rather than reimplementing it. On upload, lines are parsed once and
the `Record`s are stored structured in SQLite (`store.py`); each page view fetches a dataset's
records, builds the same `FilterSpec` predicate from query parameters, filters, and runs the same
`summarise` — so the web view-model is `web.py` turning a `Summary` into level bars, a bucketed
timeline and template tables, with nothing in the analysis layer duplicated.

## Key design decisions

**Prefer JSON, fall back to text, never drop a line.** Each line is tried as a JSON object first and
parsed as text otherwise. A line that matches neither shape — a stack-trace frame, say — is still
returned with an unknown level rather than discarded, so `grep` still finds it. Silently losing lines
is the one thing a log tool must not do.

**Normalise levels across ecosystems.** `Information`, `INF`, `info` and the numeric `30` all map to
one `INFO` member of an ordered enum; `CRITICAL`, `PANIC` and `FTL` all map to `FATAL`. Because the
enum is ordered, `--level warn` means "at least this severe" with a single comparison. Numeric levels
follow the pino/bunyan convention.

**Group messages by template.** In `stats`, variable parts of a message — numbers, UUIDs, long hex
strings, quoted strings — are replaced with placeholders, so `Processed order 118 in 571 ms` and
`Processed order 119 in 494 ms` count as one. That is what you want when asking what a service is
mostly saying.

**Exit codes for scripting.** `0` when something matched, `1` when nothing did, `2` for a usage or
file error. That makes `logsift build.log --level error && fail-the-build` work in CI.

## Data and interface

**Record.** `timestamp` (aware datetime or none), `level` (normalised enum), `message`, `fields` (the
remaining structured keys), `source` (file name), `raw` (the original line).

**CLI.**

```
logsift [filter] FILES...   print matching lines (default command)
logsift stats   FILES...    summarise matching lines

Filters (both modes):
  --level L            minimum severity          --only L,L     exact levels
  --since T --until T  ISO time or duration (15m) --where k=v    structured field equals value
  -g / --grep REGEX    keep (repeatable, AND)    -v / --exclude drop   -i  ignore case

filter: --format pretty|raw|json   --limit N
stats:  --bucket 5m   --top N   --json
```

Files may be plain or `.gz`; with no files it reads stdin, so `kubectl logs ... | logsift` works.

## Testing

Seventy-seven tests. They cover the parsers against real line shapes from several logging ecosystems
(Serilog compact keys, pino numeric levels and epoch-millisecond timestamps, Python logging, syslog,
bracketed and colon-terminated level tokens, trailing key=value pairs), the filter predicates
including durations and nested `--where` keys, the summary maths (level counts, message templating,
time-bucket boundaries), and the CLI end to end against sample files in the repo — multiple files,
gzip, stdin, output formats and exit codes. The console adds tests for the SQLite store (parsed
records round-tripping with their level, timestamp and fields intact) and the FastAPI app via
`TestClient` (upload, the level/text/field filters narrowing the summary, the JSON API, and errors).
The matrix runs it on Linux and Windows across four Python versions.

## Running and deployment

`pip install git+https://github.com/BoogeyCodes/logsift.git` gives a `logsift` command. The console
is `pip install "logsift[web]"` then `logsift serve`, or `docker run` the included image (datasets
persist in a mounted `/data` volume). For development, `pip install -e ".[dev]"`, then `ruff check .`
and `pytest`.

## Challenges and lessons

**Telling a level token from an ordinary word.** An early version matched any level word anywhere,
so a message like "Loaded INFORMATION_SCHEMA" was misread as INFO. The fix was to require a bare token
to be upper-case and standalone, while still accepting decorated tokens like `[info]` or `error:` in
any case. Parsing human-written log text is full of these ambiguities, and tests against real lines
are the only way to stay honest.

**Timestamps in the wild.** ISO-8601 with `Z` or an offset, space-separated with comma milliseconds,
epoch seconds, epoch milliseconds, and syslog's month-name form all appear. Each is handled, and
anything unrecognised returns none rather than raising, so one odd line never stops the stream.

## Future work

- `--follow` to tail a growing file.
- `--count-by <field>` for a histogram of a field's values.
- A `--format csv` for spreadsheets.

## Links

- Repository: https://github.com/BoogeyCodes/logsift
