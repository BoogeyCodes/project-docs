# ByteFmt

> A .NET `DataSize` value type that parses and formats human byte sizes in binary (KiB) and decimal
> (KB) units, with comparison and arithmetic.

- **Repository:** https://github.com/BoogeyCodes/bytefmt
- **Status:** Complete
- **Primary stack:** C# / .NET 8 (library, NuGet-packable)
- **Last updated:** 2026-11-03

## Overview

ByteFmt is a small .NET library whose one type, `DataSize`, represents a quantity of bytes and knows
how to parse and format itself. It reads human strings like `"1.5 GiB"`, `"900 MB"` and `"1,024 KB"`,
formats a byte count back into the largest sensible unit, and distinguishes binary units (KiB = 1024)
from decimal units (KB = 1000). `DataSize` is an immutable value type with comparison and arithmetic,
so it sorts and adds like a number.

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
whole bytes.

## Testing

Forty-one tests. Theory-driven cases cover binary and decimal formatting at the unit boundaries
(1023 B, 1024 → 1 KiB, …), precision, and parsing of every suffix form including bare letters, mixed
case and thousands separators; plus round-tripping through humanize and parse, the comparison and
arithmetic operators, equality and hashing, and rejected input (empty, non-numeric, negative, unknown
suffix). CI runs on Linux and Windows and builds the NuGet package.

## Running and deployment

`dotnet test` to run the suite; `dotnet pack` to build the package. It is a library referenced by
other projects.

## Challenges and lessons

**The binary/decimal ambiguity is the whole point, so make it explicit.** The temptation is to pick
one convention and hide it; the better design surfaces it — binary by default for formatting, inferred
from the suffix for parsing — and documents the one judgement call (a bare `1K` is binary). Theory
tests across both standards at the unit boundaries kept the two code paths honest.

## Future work

- `IFormattable` and custom format strings (`"{0:MiB}"`).
- Subtraction and division (a ratio between two sizes).
- A `System.Text.Json` converter.

## Links

- Repository: https://github.com/BoogeyCodes/bytefmt
- Related docs: [jsondiff.md](jsondiff.md) (another small, well-tested .NET project)
