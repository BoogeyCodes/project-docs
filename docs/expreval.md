# expreval

> A safe arithmetic expression evaluator for Python — without `eval`.

- **Repository:** https://github.com/BoogeyCodes/expreval
- **Status:** Complete
- **Primary stack:** Python 3.10+ (standard library only)
- **Last updated:** 2026-11-03

## Overview

expreval evaluates a mathematical expression from a string safely: a tokenizer feeds a shunting-yard
parser, and the resulting postfix is evaluated. It supports operators, parentheses, functions,
constants and variables — and nothing else, so it cannot run arbitrary code the way `eval` can.

## Purpose and goals

Evaluating a user-supplied formula with Python's `eval` is a security hole. The goal was a small,
dependency-free evaluator that computes the arithmetic itself, handles the real parsing concerns
(precedence, associativity, unary minus, function arguments), and reports every failure as a clear
`ExprError`.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | Python 3.10+ | Clear, and `math` provides the functions |
| Dependencies | None | Safe and self-contained |
| Tests | pytest | Arithmetic, functions, variables, errors, CLI |
| CI | GitHub Actions matrix | Python 3.10–3.13 |

## Architecture

```
  string -> tokenize -> shunting-yard parse (-> postfix) -> stack evaluation -> float
```

## Key design decisions

**Shunting-yard, not `eval`.** Tokens are reordered into postfix by the shunting-yard algorithm,
respecting precedence and the right-associativity of `^`, then evaluated on a stack. The only
operations available are the ones built in, so a formula cannot reach outside arithmetic.

**Unary minus bound correctly.** Unary minus is given a precedence between `*` and `^`, so `-2^2` is
`-(2^2) = -4`, matching mathematical convention, and the parser distinguishes unary from binary minus
by what precedes it.

**Function argument counting in the parser.** The parser tracks how many comma-separated arguments a
call has, so `max(1, 2, 3)` and the optional-argument `log(x, base)` and `round(x, n)` all work, with
arity checked at evaluation.

**Every failure is an ExprError.** Unbalanced parentheses, unknown names, wrong argument counts,
division by zero and math domain errors all raise one exception type with a message, never a bare
crash.

## Testing

Twenty-six tests covering precedence and associativity, unary minus, scientific notation, constants,
single/variadic/optional-argument functions, nested calls, variables, and every error path, plus the
CLI. CI runs on Python 3.10–3.13.

## Challenges and lessons

**`round` needs an integer precision.** Passing a float second argument to Python's `round` raises a
`TypeError`; the fix was to coerce it, and to wrap such library errors as `ExprError` so the evaluator
never leaks an unexpected exception type.

## Future work

- Boolean and comparison operators.
- A pluggable function registry.
- Exact arithmetic via `decimal`/`fractions`.

## Links

- Repository: https://github.com/BoogeyCodes/expreval
- Related docs: [gridcalc.md](gridcalc.md) (a formula parser in TypeScript)
