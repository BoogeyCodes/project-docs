# ByteFmt

> A .NET `DataSize` value type that parses and formats human byte sizes — and **DiskScope**, a
> disk-usage explorer built on it (scan engine + ASP.NET Core dashboard and REST API).

- **Repository:** https://github.com/BoogeyCodes/bytefmt
- **Status:** Complete
- **Primary stack:** C# / .NET 8 — a NuGet-packable library plus an ASP.NET Core 8 web app
- **Last updated:** 2026-11-13

## Overview

ByteFmt began as a small .NET library whose one type, `DataSize`, represents a quantity of bytes and
knows how to parse and format itself — reading human strings like `"1.5 GiB"`, `"900 MB"`, `"1,024 KB"`,
distinguishing binary units (KiB = 1024) from decimal (KB = 1000), and behaving as a proper immutable
value type with comparison and arithmetic.

On top of that primitive sits **DiskScope**, a disk-usage explorer. A scan engine (`ByteFmt.DiskScan`)
walks a directory tree and reports where the space goes — total size, the biggest immediate folders
(sized recursively), a breakdown by file extension, and the largest individual files — and an ASP.NET
Core 8 web app (`ByteFmt.Web`) presents it as a dashboard and a JSON REST API. Every size in DiskScope
is a `DataSize`, so the library's formatting is what makes the analyzer readable.

## Purpose and goals

"Is it KB or KiB? 1000 or 1024?" is a question projects get wrong constantly, and byte formatting is
the kind of helper that gets reimplemented slightly differently in every codebase. The goal was a
single, correct, well-tested value type that makes the binary/decimal distinction explicit and
behaves like a proper .NET value — equatable, comparable, immutable.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language / runtime | C# / .NET 8 | A `readonly struct` is the natural fit for a value type |
| Packaging | NuGet (`dotnet pack`) | Reusable as a dependency |
| Tests | xUnit | Theory-driven tests across units and suffix forms |
| CI | GitHub Actions matrix | Linux and Windows; also packs the package |

## Design and key decisions

**Bytes are the single source of truth.** A `DataSize` stores a `long` byte count and nothing else;
every unit value (`Kibibytes`, `Mebibytes`, …) is derived. There is no second representation to drift
out of sync, so equality and hashing are exact and keyed on the byte count.

**Binary vs decimal is a decision at the edges, not in storage.** The stored value is unit-agnostic.
When formatting you pass a `UnitStandard` (binary or decimal); when parsing, the standard is inferred
from the suffix — `KiB` is binary, `KB` is decimal, and a bare `K`/`M`/`G` is treated as binary
because that is what people usually mean by "1K".

**A real value type.** `DataSize` is a `readonly struct` implementing `IEquatable` and `IComparable`,
with `==`, `<`, `>`, `+` and `* int` operators, so it sorts with `Array.Sort`, compares naturally and
composes like a number — rather than being a bag of static helper methods.

**Fail loudly on bad values.** Negative sizes throw `ArgumentOutOfRangeException`; parsing and
arithmetic use `checked` arithmetic so an overflow throws (or, via `TryParse`, returns `false`) rather
than silently wrapping. `Parse` throws `FormatException` on bad input; `TryParse` is the non-throwing
counterpart.

## Data and interface

```csharp
new DataSize(long bytes)
DataSize.FromBytes/FromKibibytes/FromMebibytes/FromGibibytes/FromTebibytes(...)
size.Bytes / .Kibibytes / .Mebibytes / .Gibibytes / .Tebibytes
size.Humanize(UnitStandard standard = Binary, int decimals = 2)  // "1.5 KiB"
size.ToString()                                                   // binary units
DataSize.Parse(string) / DataSize.TryParse(string, out DataSize)
operators: + , * int, < > <= >= == != , CompareTo, Equals, GetHashCode
```

Formatting picks the largest unit that keeps the value at or above 1, and shows sub-unit values as
whole bytes. `FractionOf(whole)` gives a size as a fraction of another, for share/percentage views.

## DiskScope: the analyzer built on the value type

**A separate, testable scan engine.** `ByteFmt.DiskScan.DirectoryScanner` walks the tree in a single
recursive pass and returns a `ScanResult` record — total `DataSize`, file/directory counts, the top
immediate folders (each sized recursively), per-extension aggregates, and the largest files. It is a
plain library with no web dependency, so it is unit-tested against a temp directory tree with known
sizes. Inaccessible files/folders are caught and counted (`SkippedCount`) rather than aborting the
scan, and the largest-files list is kept bounded (a little more than `Top`, trimmed) so scanning a
tree with millions of files does not hold them all in memory.

**A thin web layer.** `ByteFmt.Web` is an ASP.NET Core 8 minimal API: `GET /` and `GET /scan` render
the dashboard (server-side HTML, no templating engine), `GET /api/scan` returns the `ScanResult` as
JSON with both raw bytes and humanised strings (binary or decimal units), and `GET /healthz` is a
probe. A `ScanPresenter` maps the domain result to a response DTO, computing each row's share with
`DataSize.FractionOf`. The app is integration-tested with `WebApplicationFactory` (the scan API, unit
switching, error cases, and the rendered pages).

**Honest about the trust boundary.** The scanner reads the server's own filesystem, so the docs and
the UI say to bind it to localhost or run it in a container with a read-only mount — not to expose it
to untrusted networks.

## Testing

Sixty-two tests across three projects. Theory-driven cases cover binary and decimal formatting at the unit boundaries
(1023 B, 1024 → 1 KiB, …), precision, and parsing of every suffix form including bare letters, mixed
case and thousands separators; plus round-tripping through humanize and parse, the comparison and
arithmetic operators, equality and hashing, and rejected input (empty, non-numeric, negative, unknown
suffix). The `DirectoryScanner` is tested over a temp tree with known file sizes (totals, recursive
folder sizes, extension aggregation, largest files, the `Top` limit, and empty/missing/blank-path
handling), and the web API with `WebApplicationFactory`. CI runs on Linux and Windows and builds the
NuGet package.

## Running and deployment

`dotnet test` runs the whole suite; `dotnet pack src/ByteFmt` builds the library package;
`dotnet run --project src/ByteFmt.Web` (or the `Dockerfile`) serves the DiskScope dashboard. The
library is referenced by other projects; the analyzer is a standalone app.

## Challenges and lessons

**The binary/decimal ambiguity is the whole point, so make it explicit.** The temptation is to pick
one convention and hide it; the better design surfaces it — binary by default for formatting, inferred
from the suffix for parsing — and documents the one judgement call (a bare `1K` is binary). Theory
tests across both standards at the unit boundaries kept the two code paths honest.

## Future work

- `IFormattable` and custom format strings (`"{0:MiB}"`).
- A `System.Text.Json` converter for `DataSize`.
- DiskScope: an interactive treemap, and scan history/comparison.

## Links

- Repository: https://github.com/BoogeyCodes/bytefmt
- Related docs: [jsondiff.md](jsondiff.md) (another small, well-tested .NET project)
