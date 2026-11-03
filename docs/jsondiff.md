# jsondiff

> A command-line tool that structurally compares two JSON files and reports what was added, removed
> and changed, each located by a precise path.

- **Repository:** https://github.com/BoogeyCodes/jsondiff
- **Status:** Complete
- **Primary stack:** C# / .NET 8 (console tool + library)
- **Last updated:** 2026-11-03

## Overview

jsondiff compares two JSON documents by structure rather than by text. Where a plain `diff` reports
indentation, trailing commas and reordered keys as differences, jsondiff parses both files and walks
the trees, so it reports only real differences and points at each one with a JSON-Pointer-style path
such as `/database/host` or `/features/2`. It prints a readable summary by default or machine-readable
JSON on request, and its exit code reflects whether the files differ, so it slots into scripts and CI.

It ships as two pieces: a comparison library with no command-line concerns, and a thin CLI over it.
That separation is the point of the project — it is what makes the comparison logic exhaustively
testable and the CLI itself driveable in tests.

## Purpose and goals

Comparing JSON with text tools is frustrating: formatting noise drowns the real change, and you still
have to hunt for *where* it is. The goal was a small, correct tool that answers "what actually changed
between these two JSON documents, and where?" and is usable both by a human reading the output and by
a script checking an exit code.

Goals:

- Compare by data, ignoring formatting and key order.
- Address every difference with an unambiguous path.
- Treat equal numbers as equal regardless of spelling (`1`, `1.0`, `1e0`).
- Be scriptable: a meaningful exit code and a JSON output mode.
- Keep the comparison logic independent of the CLI so it can be tested on its own.

Non-goals: it is not a patch/merge tool and does not produce an edit script; it reports differences,
not how to apply them.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language / runtime | C# / .NET 8 | Fast startup, `System.Text.Json` in the box |
| JSON | `System.Text.Json` (`JsonDocument`) | Allocation-light read-only tree traversal |
| Packaging | .NET tool (`PackAsTool`) | `dotnet tool install --global jsondiff-cli` |
| Tests | xUnit | Library and in-process CLI tests |
| CI | GitHub Actions matrix | Linux and Windows; also packs the tool |

## Architecture

```
  CLI (src/JsonDiff/Program.cs)
    parse args · read files · pick text/JSON output · set exit code
        │ calls
        ▼
  JsonComparer.Compare(left, right, options)   (src/JsonDiff.Core)
    recursive walk:
      objects -> match by key (added / removed / recurse)
      arrays  -> by index, or as a multiset with --ignore-array-order
      scalars -> by type and numeric/string value
        │ produces
        ▼
  IReadOnlyList<Change>  { kind, path, oldValue, newValue }
        │ rendered by
        ▼
  DiffReport.ToText / ToJson
```

The library knows nothing about files, arguments or exit codes; the CLI supplies those and is a thin
shell. The `Program` class is exposed to the test project so arg parsing and exit codes are tested in
process.

## Key design decisions

**Library and CLI are separate projects.** All comparison logic lives in `JsonDiff.Core` with no
command-line dependency. This is what makes the comparer unit-testable in isolation and lets the CLI
be driven directly in tests (passing `args` and capturing stdout/stderr and the exit code), rather
than shelling out to a built binary.

**Compare by data, not text.** Objects are matched by key and arrays by index, so reordered object
keys are never a difference and a changed array element reports its position. Numbers are compared
numerically, so `1` and `1.0` are equal — a distinction a text diff cannot make.

**Paths use JSON Pointer escaping.** Each difference is addressed like `/user/roles/0`, with `/` and
`~` inside a key escaped per RFC 6901, so the path is always unambiguous even for awkward keys.

**Array order is configurable.** By default arrays compare by position, which also surfaces
reordering; `--ignore-array-order` compares them as multisets for cases where order is not meaningful.
Making this a flag keeps the default honest while serving both needs.

**Exit codes are part of the contract.** `0` equal, `1` different, `2` usage/input error — the same
convention as `diff` — so `jsondiff expected.json actual.json` can gate a CI step.

## Data and interface

**Change:** `Kind` (Added / Removed / Changed), `Path`, `OldValue`, `NewValue`.

**CLI.**

```
jsondiff <left.json> <right.json> [--json] [--ignore-array-order] [-h|--help]
```

Text output lists each change (`+`, `-`, `~`) and a summary line; `--json` emits an array of change
objects for piping into `jq` or asserting on in tests.

## Testing

Twenty-seven tests. The comparer is tested against scalars, nested objects, arrays by index and as
multisets, numeric equality, type changes, key additions/removals and JSON-Pointer escaping; the
reports in both text and JSON form; and the CLI end to end for every exit code and error path (equal,
different, `--json`, `--ignore-array-order`, wrong argument count, unknown option, missing file,
invalid JSON, and help). CI runs the suite on Linux and Windows and packs the tool as a NuGet
artifact.

## Running and deployment

`dotnet tool install --global jsondiff-cli` installs the `jsondiff` command, or build from source and
`dotnet run --project src/JsonDiff -- a.json b.json`. There is no service to host; distribution is the
tool package.

## Challenges and lessons

**Canonical rendering for value equality.** Deciding whether two object or array values are "equal"
for a multiset comparison, or how to display a composite value, needs a stable, order-independent
rendering. A small recursive canonicaliser (sorting object keys, normalising numbers) solved both the
equality check and the compact display, and keeping it in one place avoided two subtly different
notions of equality.

**An XML comment broke the build.** A `<!-- ... --global ... -->` comment in the project file failed
to parse, because `--` cannot appear inside an XML comment. A small thing, but a good reminder that
the build file is XML with its own rules, not free text.

## Future work

- A `--only added,changed` filter to narrow the output.
- Reading a document from stdin so it fits in a pipeline.
- A side-by-side coloured view for terminals.

## Links

- Repository: https://github.com/BoogeyCodes/jsondiff
- Related docs: [logsift.md](logsift.md) (another small, scriptable CLI with meaningful exit codes)
