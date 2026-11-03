# Outcome

> Lightweight Result and Option functional types for C#, with Map, Bind and Match.

- **Repository:** https://github.com/BoogeyCodes/outcome
- **Status:** Complete
- **Primary stack:** C# / .NET 8 (library, NuGet-packable)
- **Last updated:** 2026-11-03

## Overview

Outcome provides two types: `Option<T>` for a value that may be absent, and `Result<T, TError>` for an
operation that may fail with a typed error. Both come with `Map`, `Bind` and `Match` so operations
compose, and a missing value or an error short-circuits the rest of a chain. It is a small, focused
implementation of the "railway-oriented" pattern.

## Purpose and goals

`null` and exceptions are the usual ways to express "might be missing" and "might fail", and both are
easy to misuse: a `null` slips past the compiler, and exceptions for expected failures are heavy and
easy to forget. The goal was to put both cases in the type system, so the compiler forces you to
handle them, in a couple of small structs.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language / runtime | C# / .NET 8 | `readonly struct` fits value types |
| Packaging | NuGet | Reusable as a dependency |
| Tests | xUnit | Combinators, conversions, equality |
| CI | GitHub Actions matrix | Linux/Windows; packs the package |

## Key design decisions

**Immutable value types.** Both are `readonly struct`s with value equality, so they compare and behave
like the values they wrap rather than reference objects.

**The combinator set.** `Map` transforms the contained value; `Bind` chains an operation that itself
returns the type (errors/absence short-circuit); `Match` collapses both cases into a result. `Result`
adds `MapErr` and `ToOption`; `Option` adds `Where` and `ToResult`.

**Bridges to the rest of C#.** `Option.From` turns a nullable (reference or value) into an Option;
`TryGetValue` follows the familiar `out` pattern; implicit conversion lets a value become `Some`; and
`Result.Try` captures a thrown exception as an error, so exception-based APIs fold into the type.

## Testing

Eighteen tests across construction, `Match`, `Map`/`Bind`/`MapErr`, filtering, fallbacks,
null-bridging and implicit conversion, `Try`, the `Option`/`Result` conversions, `TryGetValue`, value
equality, and a composed pipeline. CI runs on Linux and Windows and builds the NuGet package.

## Future work

- Async combinators (`MapAsync`, `BindAsync`).
- LINQ query syntax (`Select`/`SelectMany`).

## Links

- Repository: https://github.com/BoogeyCodes/outcome
