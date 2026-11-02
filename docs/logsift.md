# logsift

> A zero-dependency command-line tool that filters and summarises JSON or plain-text application logs.

- **Repository:** https://github.com/BoogeyCodes/logsift
- **Status:** Complete
- **Primary stack:** Python 3.10+ (standard library only)
- **Last updated:** 2026-11-02

## Overview

logsift reads log files — or standard input — understands both structured JSON-lines output and
classic `timestamp LEVEL message` text, and lets you ask questions of them from the terminal. Two
modes: `filter` prints the matching lines, and `stats` summarises them. It can answer "what errored in
the last fifteen minutes?" or "which messages dominate this file?" without a `jq` incantation, and it
streams, so multi-gigabyte files are fine.

It exists for the gap between a raw log file and a log platform. A lot of debugging starts with a
`kubectl logs` dump or a `.log.gz` pulled off a server — files that never reached a search index.
logsift is the small, dependency-free tool for exactly that moment.

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
| Dependencies | None (standard library only) | Installs and runs anywhere Python does; nothing to audit |
| CLI | `argparse` with subcommands | Familiar interface, built in |
| Packaging | `pyproject.toml`, console entry point | `pip install` gives a `logsift` command |
| Lint | ruff | Fast, strict, zero-config |
| Testing | pytest | The suite that proves the parsers against real shapes |
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

Sixty-four tests. They cover the parsers against real line shapes from several logging ecosystems
(Serilog compact keys, pino numeric levels and epoch-millisecond timestamps, Python logging, syslog,
bracketed and colon-terminated level tokens, trailing key=value pairs), the filter predicates
including durations and nested `--where` keys, the summary maths (level counts, message templating,
time-bucket boundaries), and the CLI end to end against sample files in the repo — multiple files,
gzip, stdin, output formats and exit codes. The matrix runs it on Linux and Windows across four Python
versions.

## Running and deployment

`pip install git+https://github.com/BoogeyCodes/logsift.git` gives a `logsift` command. For
development, `pip install -e ".[dev]"`, then `ruff check .` and `pytest`. There is no runtime
deployment — it is a tool you install and run.

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
