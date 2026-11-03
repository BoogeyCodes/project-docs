# gridcalc

> A zero-dependency spreadsheet formula engine in TypeScript: a formula parser, cell references and
> ranges, a dependency graph with incremental recompute, and circular-reference detection.

- **Repository:** https://github.com/BoogeyCodes/gridcalc
- **Status:** Complete
- **Primary stack:** TypeScript (standard library only), built with tsup, tested with Vitest
- **Last updated:** 2026-11-03

## Overview

gridcalc is the computing core of a spreadsheet, without the user interface. You set cells to
numbers, text or formulas — strings beginning with `=` that reference other cells — and read back
their computed values. It parses formulas with a real lexer and Pratt parser, understands A1-style
references and ranges, evaluates with a set of built-in functions, recomputes only the cells affected
by a change, and detects circular references. It is a library of a few hundred readable lines, meant
to show the machinery of a spreadsheet rather than to replace Excel.

## Purpose and goals

A spreadsheet is a small but complete computer: a little language to parse, a dependency graph to
evaluate in the right order, and error values that propagate through it. That combination is a great
way to show parsing, evaluation and reactive computation in one self-contained project.

Goals:

- Parse formulas correctly, including operator precedence and associativity.
- Resolve references and ranges, and evaluate a useful set of functions.
- Recompute efficiently: changing one cell should recompute only what depends on it.
- Detect circular references and report them as an error rather than looping forever.
- Keep it a dependency-free library that ships as both ESM and CommonJS.

Non-goals: a UI, file formats (xlsx/csv), absolute references and copy-adjustment, and the long tail
of Excel functions. These are noted as future work.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | TypeScript 5 | Types make the AST and value model precise |
| Dependencies | None | A small, self-contained engine |
| Build | tsup (esbuild) | ESM, CJS and `.d.ts` from one config |
| Tests | Vitest | Fast, TypeScript-native |
| CI | GitHub Actions matrix | Linux and Windows, Node 18/20/22 |

## Architecture

```
  Grid.set('C3', '=SUM(C1:C2)')
        │
        ▼
  tokenize (lexer)  ─►  parseFormula (Pratt parser)  ─►  AST
        │                                                 │ collectReferences
        │                                                 ▼
        │                                  dependency graph: precedents ⇄ dependents
        ▼
  Grid.get('C3')
        │  lazy, memoised
        ▼
  evaluate(AST, resolve)   resolve(ref) = Grid.get(ref), recursively
        │  if a cell is already on the evaluation stack ⇒ #CIRCULAR!
        ▼
  Value, cached until a precedent changes
```

The pieces are deliberately separable: the lexer, parser, evaluator and reference helpers each stand
alone and are exported, and the `Grid` ties them together with the dependency graph and cache.

## Key design decisions

**A Pratt parser for formulas.** Operator-precedence parsing handles the precedence table and the
right-associativity of `^` cleanly, without a tangle of grammar rules. Unary minus is given a binding
power higher than `^`, so `=-2^2` is `4` — the spreadsheet convention, which differs from most
programming languages. This is a deliberate, documented choice and a test pins it.

**Cell references versus function names, decided in the lexer.** `A1` and `SUM` both start with
letters. The lexer distinguishes them by a simple rule: letters immediately followed by digits are a
cell reference; letters alone are an identifier (a function name, or `TRUE`/`FALSE`). This keeps the
parser simple.

**Lazy, memoised evaluation.** `get(cell)` evaluates on demand and caches the result; references are
resolved by recursively getting other cells. The cache is what makes repeated reads cheap.

**A dependency graph purely for invalidation.** Each formula's referenced cells are recorded as its
precedents, and the inverse map gives each cell its dependents. Setting a cell clears the cache for
that cell and everything that transitively depends on it, and nothing else — so a change recomputes
the minimum. This is the "incremental" part, and it is the same idea a real spreadsheet uses.

**Circular references caught during evaluation.** Rather than analysing the graph for cycles up
front, the evaluator keeps the set of cells currently being computed; re-entering one means a cycle,
and those cells resolve to `#CIRCULAR!`. Breaking the cycle and re-reading recovers, because the
cache was invalidated. This is simple and always correct, even for cycles formed across several edits.

**Errors are values that propagate.** `#DIV/0!`, `#VALUE!`, `#NAME?` and `#CIRCULAR!` are ordinary
cell values; any operator or function that receives one passes it along, so an error surfaces
wherever it is eventually read.

## Data and interface

**Grid API.**

```ts
grid.set(ref, value)     // number | string | boolean; a string starting with '=' is a formula
grid.setMany({ A1: 1 })
grid.get(ref)            // Value = number | string | boolean | CellError; unset cells read as ''
grid.getFormula(ref)     // the raw '=...' text, or undefined
grid.delete(ref)
```

**Functions:** `SUM`, `AVERAGE`, `MIN`, `MAX`, `PRODUCT`, `COUNT`, `IF`, `AND`, `OR`, `NOT`, `ABS`,
`SQRT`, `INT`, `ROUND`, `MOD`, `CONCAT`, `LEN`, `UPPER`, `LOWER`, `TRIM`.

**Semantics:** an unset cell is `0` in arithmetic and `""` in text; aggregation ignores non-numeric
text and propagates errors; text comparison is case-insensitive.

## Testing

Twenty-six tests across four files: the reference helpers (column/index conversion round-trips, range
expansion), the parser (precedence, right-associative `^`, refs/ranges/calls, malformed input), the
evaluator (arithmetic, comparisons, concatenation, every function family, error propagation), and the
grid (cross-cell references, incremental recompute down a 20-cell chain, direct and indirect cycles,
and recovery once a cycle is broken). CI runs them on Linux and Windows across Node 18, 20 and 22.

## Running and deployment

`npm install` then `npm test`, `npm run typecheck` and `npm run build` (ESM + CJS + types via tsup).
It is a library, consumed as a dependency; there is nothing to deploy.

## Challenges and lessons

**The precedence of unary minus.** `=-2^2` is `4` in spreadsheets but `-4` in Python and many
languages, because spreadsheets bind unary minus tighter than exponentiation. My first implementation
followed the programming-language convention and a test caught the mismatch; fixing it was a
one-number change to a binding power, but deciding *which* convention to follow — and documenting it —
was the real work. For a spreadsheet engine, matching spreadsheets is the correct call.

**Invalidation, not recomputation, is the hard part.** Computing a formula is easy; knowing exactly
which cached values a change makes stale is where the interesting work is. Keeping an explicit
precedent/dependent graph and invalidating transitively made this correct and kept recomputation
minimal.

## Future work

- `$A$1` absolute references and relative-reference adjustment when formulas are copied.
- More functions (`VLOOKUP`, `SUMIF`, date math).
- A `topologicalOrder()` for batch recalculation, and sheet serialisation.

## Links

- Repository: https://github.com/BoogeyCodes/gridcalc
- Related docs: [jsondiff.md](jsondiff.md) (another parser/tree-walking project)
