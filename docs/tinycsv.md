# tinycsv

> A tiny, zero-dependency RFC 4180 CSV parser and stringifier for TypeScript.

- **Repository:** https://github.com/BoogeyCodes/tinycsv
- **Status:** Complete
- **Primary stack:** TypeScript (standard library only), tsup, Vitest
- **Last updated:** 2026-11-03

## Overview

tinycsv parses and serialises CSV following RFC 4180, so quoted fields, escaped quotes, embedded
commas and newlines all work — the cases a naive `split(',')` gets wrong. It also maps to and from
arrays of objects, reads TSV via a delimiter option, and is round-trip safe.

## Purpose and goals

CSV looks trivial until a field contains a comma, a quote or a newline. The goal was a small, correct
parser and a matching stringifier that quotes only what needs quoting, with no dependencies.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | TypeScript 5 | Typed API |
| Dependencies | None | Small and self-contained |
| Build | tsup | ESM + CJS + types |
| Tests | Vitest | Parse, stringify, round-trip |
| CI | GitHub Actions matrix | Linux/Windows, Node 18/20/22 |

## Key design decisions

**A state-machine parser, not `split`.** Parsing walks the input character by character, tracking
whether it is inside a quoted field, so a delimiter, newline or `""` escape inside quotes is handled
correctly. Both `\n`, `\r\n` and bare `\r` endings are accepted.

**Stringify quotes only what needs it.** A field is quoted only if it contains the delimiter, a quote
or a newline (or always, on request), and `null`/`undefined` render as empty — so the output is
minimal and `parse(stringify(x))` returns the original data, which a test asserts.

**Object mapping on top.** `parseObjects` keys rows by a header; `stringifyObjects` writes a header
and rows, inferring columns or taking an explicit order.

## Testing

Nineteen tests covering quoted fields with commas and newlines, doubled-quote escapes, CRLF/CR
endings, empty fields and lines, custom delimiters (TSV), trimming, trailing fields, object mapping,
selective quoting, and a parse/stringify round trip. CI runs on Linux and Windows across Node
18/20/22.

## Future work

- Streaming parse for large files.
- Type coercion hooks (numbers, booleans, dates).

## Links

- Repository: https://github.com/BoogeyCodes/tinycsv
- Related docs: [tabulate.md](tabulate.md) (a CSV-to-table CLI in .NET)
