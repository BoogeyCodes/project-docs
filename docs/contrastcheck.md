# Contrast Check

> A WCAG colour-contrast checker built with React and TypeScript.

- **Repository:** https://github.com/BoogeyCodes/contrastcheck
- **Status:** Complete
- **Primary stack:** React 19, TypeScript, Vite, Vitest
- **Last updated:** 2026-11-03

## Overview

Contrast Check is a small single-page app for checking whether two colours have enough contrast to be
readable. You enter a foreground and a background colour and immediately see the contrast ratio, a
live preview of sample text, and a pass/fail for every WCAG criterion — and when a pair fails, the
nearest foreground colour that would pass. The accessibility maths is a pure, well-tested module; the
React UI is a thin layer over it.

## Purpose and goals

Readable colour is an accessibility requirement: WCAG sets minimum contrast ratios for text and UI
components, and failing them locks out people with low vision. The goal was an instant checker — type
two colours, see the ratios and the verdicts — that also shows the contrast calculation as real,
inspectable code rather than a black box, and that fixes a failing pair with one click.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| UI | React 19 + TypeScript | Typed components and hooks |
| Build/dev | Vite | Fast dev server and build |
| Tests | Vitest + Testing Library | Unit tests for the maths, component tests for the UI |
| Dependencies | None beyond React | The contrast logic is hand-written, not a library |

## Architecture

```
  src/lib/contrast.ts     pure WCAG maths (no React)
    parseHex · relativeLuminance · contrastRatio · wcag · suggestForeground
        │ used by
        ▼
  src/App.tsx             state via hooks; useMemo for the derived result and suggestion
    ColorField · preview · level breakdown · suggestion chip
```

The split is deliberate: all the maths lives in a framework-free module, and the component only wires
inputs to it and renders the output.

## Key design decisions

**The maths is a separate, pure module.** `contrast.ts` has no React in it, so it can be unit-tested
against known reference values (black on white is exactly `21:1`, `#777` on white is `≈ 4.48`) and
reasoned about independently. This is the part that has to be correct, so it is isolated.

**Implement WCAG to the letter.** Relative luminance gamma-expands each sRGB channel
(`c/12.92` below the threshold, `((c+0.055)/1.055)^2.4` above) and takes the weighted sum
`0.2126 R + 0.7152 G + 0.0722 B`; the ratio is `(L_lighter + 0.05)/(L_darker + 0.05)`. These are the
exact definitions, so the numbers match any compliant checker.

**Suggestions round before they test.** The "nearest passing colour" marches the foreground toward
black or white, but it rounds each candidate to an integer colour *before* checking its ratio, so the
colour it returns genuinely meets the target — an earlier version returned a float that passed and an
integer that did not. It then prefers whichever direction changes the colour least.

**Derived state via `useMemo`.** The result and the suggestion are computed from the two colours with
memoised selectors, so they recompute only when a colour changes.

## Data and interface

The UI takes two hex strings (`#rgb` or `#rrggbb`) and shows: the ratio; AA/AAA for normal and large
text and the UI-component threshold; a preview; and, on failure, a suggested colour. The core module
exports `parseHex`, `toHex`, `relativeLuminance`, `contrastRatio`, `wcag`, `formatRatio` and
`suggestForeground`.

## Testing

Seventeen tests. The contrast module is tested for hex parsing (including `#rgb`, whitespace and bad
input), luminance endpoints, ratio symmetry and a reference value, the WCAG thresholds, and that a
suggested colour actually meets its target against both light and dark backgrounds. Component tests
drive the UI with Testing Library: the default colours' verdicts, choosing a preset, and the error
state for invalid input. CI runs type-check, tests and build.

## Running and deployment

`npm install`, then `npm run dev` or `npm run build`. It is a static app; the build in `dist/` can be
served anywhere.

## Challenges and lessons

**Rounding at the boundary.** The suggestion bug — a colour that passed as a float but failed once
rounded to 8-bit channels — is a reminder that the thing you return must be the thing you tested. The
fix was to round first, then check. A reference-value test ("`#777` on white is `≈ 4.48`") also
caught an early luminance slip immediately.

## Future work

- A palette mode that checks a whole set of colours pairwise.
- Colour-blindness simulation of the preview.
- A shareable URL that encodes the two colours.

## Links

- Repository: https://github.com/BoogeyCodes/contrastcheck
