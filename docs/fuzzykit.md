# fuzzykit

> Zero-dependency string similarity and fuzzy search for TypeScript: Levenshtein, Jaro–Winkler,
> Sørensen–Dice, and a ranked search.

- **Repository:** https://github.com/BoogeyCodes/fuzzykit
- **Status:** Complete
- **Primary stack:** TypeScript (standard library only), built with tsup, tested with Vitest
- **Last updated:** 2026-11-03

## Overview

fuzzykit measures how similar two strings are and ranks a list of candidates against a query. It
provides the three similarity measures used most often — Levenshtein edit distance, Jaro–Winkler, and
the Sørensen–Dice bigram coefficient — each implemented from its definition, plus a small `search`
built on top for "did you mean …?" style ranking. It has no dependencies and ships as both ESM and
CommonJS.

## Purpose and goals

Typo-tolerant filters, command suggestions, autocomplete ranking and record linkage all reduce to the
same two questions: how similar are two strings, and which candidates are closest to a query. The goal
was to provide the standard measures correctly — verified against their textbook reference values —
and a ranking function that is good enough to drop into an app, without pulling in a dependency.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | TypeScript 5 | Typed API for the measures and the search |
| Dependencies | None | Small, self-contained algorithms |
| Build | tsup (esbuild) | ESM, CJS and `.d.ts` from one config |
| Tests | Vitest | Reference-value and ranking tests |
| CI | GitHub Actions matrix | Linux and Windows, Node 18/20/22 |

## The measures

| Function | Idea | Best for |
| --- | --- | --- |
| `levenshtein` | Minimum single-character edits | general typos |
| `jaroWinkler` | Matches + transpositions + shared-prefix boost | short strings, names |
| `diceCoefficient` | Shared adjacent-letter pairs (bigrams) | longer phrases, word order |

Each similarity function returns `0–1`; `levenshtein` returns the raw distance, with
`levenshteinSimilarity` for the normalised form.

## Key design decisions

**Implement each measure from its definition, and test against reference values.** These algorithms
are easy to get subtly wrong, so each is pinned to its canonical examples:
`jaro('MARTHA','MARHTA') ≈ 0.944`, `levenshtein('kitten','sitting') = 3`,
`dice('night','nacht') = 0.25`. If a refactor breaks one, a test says so immediately.

**Levenshtein in two rows.** The edit distance uses the standard dynamic-programming recurrence but
keeps only the previous and current rows, so memory is `O(min(a, b))` instead of the full matrix —
the usual optimisation, and worth doing for long inputs.

**Dice counts bigrams by multiplicity.** A repeated bigram is matched as many times as it appears in
both strings, not merely by presence, which is the correct Sørensen–Dice definition and matters for
strings with repeated letters.

**A substring match scores a perfect 1.** In `search`, a case-insensitive substring match is treated
as a perfect score, so exact and prefix matches always rank above approximate ones — which is what a
user expects from an autocomplete.

**Pluggable algorithm and options.** `search(query, items, { key, algorithm, threshold, limit })`
ranks strings or objects by any of the three measures, with a threshold and limit. `bestMatch`
returns the single closest result or null.

## Data and interface

```ts
levenshtein(a, b) / levenshteinSimilarity(a, b)
jaro(a, b) / jaroWinkler(a, b, prefixScale?)
diceCoefficient(a, b)
search(query, items, options?) -> { item, score }[]
bestMatch(query, items, options?) -> { item, score } | null
```

## Testing

Twenty-three tests: each measure against its reference values plus symmetry, empty-string and edge
cases; and the search for typo ranking, substring scoring, thresholds and limits, object keys, case
insensitivity, algorithm selection, and the empty-result case. CI runs on Linux and Windows across
Node 18, 20 and 22.

## Running and deployment

`npm install`, then `npm test` and `npm run build` (ESM + CJS + types via tsup). It is a library,
consumed as a dependency.

## Challenges and lessons

**Reference values are the spec.** The most valuable tests here are not edge cases but the published
canonical values for each algorithm. Encoding them as tests meant the implementations could be written
confidently and refactored safely, because "correct" has a concrete, checkable definition.

## Future work

- Highlight ranges for matched substrings, for UI rendering.
- A damerau-Levenshtein variant (transpositions as a single edit).
- Tokenised matching for multi-word queries.

## Links

- Repository: https://github.com/BoogeyCodes/fuzzykit
- Related docs: [ratekit.md](ratekit.md), [gridcalc.md](gridcalc.md) (other small, tested TypeScript libraries)
