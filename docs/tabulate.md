# tabulate

> A .NET CLI that renders CSV or TSV as an aligned ASCII or Markdown table.

- **Repository:** https://github.com/BoogeyCodes/tabulate
- **Status:** Complete
- **Primary stack:** C# / .NET 8 (CLI + library)
- **Last updated:** 2026-11-03

## Overview

tabulate reads CSV or TSV — from a file or stdin — and prints an aligned table, either as an ASCII box
for the terminal or as a GitHub-flavoured Markdown table for a document, with per-column alignment. It
is split into a core library (CSV reader and renderer) and a thin CLI.

## Purpose and goals

Raw CSV is unreadable in a terminal, and hand-aligning Markdown pipes is tedious. The goal was a small
tool that parses CSV properly (quotes, embedded commas and newlines), measures the columns, and prints
a correctly aligned table in either format.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language / runtime | C# / .NET 8 | Fast startup for a CLI |
| Tests | xUnit | Reader, renderer and in-process CLI |
| CI | GitHub Actions matrix | Linux and Windows |

## Architecture

```
  stdin / file -> Csv.Parse (RFC 4180) -> rows -> TableRenderer.Render (ASCII | Markdown) -> stdout
```

The comparison/rendering logic lives in `Tabulate.Core` with no CLI concerns; the CLI parses
arguments, reads input and sets exit codes, and is driven in-process by the tests.

## Key design decisions

**Library and CLI are separate.** `Csv.Parse` and `TableRenderer.Render` know nothing about arguments
or I/O, so they are unit-tested directly, and the CLI (`Cli.Run`) is tested in process by passing args
and capturing stdout/stderr and the exit code.

**Deterministic output.** The renderer emits `\n` line endings rather than `Environment.NewLine`, so
output is byte-identical on every platform and can be asserted exactly in tests.

**Alignment in both formats.** A `l,r,c` spec pads each cell left/right/center in ASCII, and in
Markdown encodes the same with the standard `:---`, `---:`, `:---:` separator colons. Ragged rows are
padded to the widest column.

## Testing

Nineteen tests: the CSV reader (quotes, delimiters, newlines, escapes, empty fields), the renderer
(alignment, borders, the Markdown separator row, header handling, ragged rows, empty input), and the
CLI end to end (ASCII and Markdown, TSV, alignment, help, and every error path). CI runs on Linux and
Windows.

## Challenges and lessons

**Assert against real output, not guessed output.** The Markdown separator widths and the exact
spacing were easier to get right by rendering once, capturing the output, and pinning the tests to it,
rather than predicting the strings — which also surfaced a line-ending issue that the deterministic
`\n` fixed.

## Future work

- Unicode-width awareness for CJK characters.
- A `--max-width` that wraps or truncates long cells.

## Links

- Repository: https://github.com/BoogeyCodes/tabulate
- Related docs: [tinycsv.md](tinycsv.md), [jsondiff.md](jsondiff.md)
