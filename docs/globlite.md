# globlite

> A tiny, zero-dependency glob matcher for TypeScript, compiled to RegExp.

- **Repository:** https://github.com/BoogeyCodes/globlite
- **Status:** Complete
- **Primary stack:** TypeScript (standard library only), tsup, Vitest
- **Last updated:** 2026-11-03

## Overview

globlite matches file paths against glob patterns — `*`, `**`, `?`, character classes and brace
expansion — by compiling the pattern to a regular expression. It matches strings, not the file
system, so it drops in wherever you already have a list of paths.

## Purpose and goals

Matching paths against a pattern is a recurring need, and the full glob libraries are heavy. The goal
was the subset people actually use, translated to an inspectable `RegExp`, with no dependencies.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | TypeScript 5 | Typed API |
| Dependencies | None | Small and self-contained |
| Build | tsup | ESM + CJS + types |
| Tests | Vitest | Matching and brace expansion |
| CI | GitHub Actions matrix | Linux/Windows, Node 18/20/22 |

## Key design decisions

**Compile to RegExp.** A pattern is brace-expanded, then each result is translated to an anchored
regex source: `*` → `[^/]*`, `?` → `[^/]`, `**/` → `(?:[^/]*\/)*` so it spans or skips whole segments,
character classes map across, and every other character is escaped to match literally. Matching is
then a fast, inspectable `RegExp`.

**Brace expansion is separate.** `a.{js,ts}` is expanded to `['a.js', 'a.ts']` before translation,
with nested and multiple groups handled, which keeps the translator simple.

**Path-aware wildcards and dotfiles.** `*` and `?` do not cross `/`, and by default they do not match
a leading dot in a segment (so `*` does not match `.env`); `{ dot: true }` and `{ nocase: true }`
adjust this.

## Testing

Sixteen tests across brace expansion and matching: single-segment vs cross-segment wildcards, `?`,
character classes and negation, brace alternation (including nested), dotfiles, literal special
characters, case-insensitivity, pattern arrays, and `filter`. CI runs on Linux and Windows across
Node 18/20/22.

## Future work

- `extglob` patterns.
- A `\`-separator mode for Windows paths.

## Links

- Repository: https://github.com/BoogeyCodes/globlite
