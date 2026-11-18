# difflens

> A library for side-by-side, multi-run diff views of what several coding-agent runs changed — and
> DiffLens, an interactive browser app built on it.

- **Repository:** https://github.com/BoogeyCodes/difflens
- **Status:** Complete (v0.2 — added the interactive app)
- **Primary stack:** TypeScript (library: standard library only), tsup, Vitest; Vite 5 for the app
- **Last updated:** 2026-11-18

## Overview

difflens takes each agent's file changes (before/after contents), computes a line diff per file, lines
them up in a column per agent, flags the files more than one agent touched, and can render the whole
thing as a self-contained HTML page. The library is zero-dependency, ESM and CJS.

**DiffLens** is the interactive front end: a Vite + vanilla-TypeScript single-page app that imports the
library and makes the comparison explorable. Paste an `AgentRun[]` (or load the bundled sample) and it
renders one section per file with a column per agent — green inserts, red deletes, per-cell
`+added/-removed`, explicit "not changed" cells — and a summary bar with totals, a file-path filter, an
"only diverged" toggle (files changed by more than one agent), and a side-by-side/stacked switch. The
divergence points — where the runs disagree — are badged, since those are what a reviewer looks for.

## Purpose and goals

Judging which agent did best means looking at what each one actually changed — and especially where
their solutions differ. difflens is the view layer: a clean line diff per file, arranged side by side,
with overlapping files called out, as something you can read at a glance or drop into a report.

## Key design decisions

**LCS line diff.** `diffLines` uses the longest-common-subsequence dynamic program to tag each line
equal/insert/delete — simple, correct and dependency-free.

**Comparison with overlap.** `compareRuns` computes, for every file any run changed, each run's diff
(or null when untouched) and how many runs changed it, and lists the overlapping files — the places the
runs diverge.

**A safe, self-contained renderer.** `renderHtml` emits a complete dark-theme HTML page with a column
per agent and highlighted add/remove lines; all content is HTML-escaped and there are no scripts or
external assets.

**A DOM-free view model for the app.** The interactive app keeps all presentation logic — summary
totals, per-file row views, and the filter — in a `web/model.ts` module that takes a `Comparison` and
returns plain data (`buildViewModel`, `summarize`, `toFileView`, `filterFiles`). It touches no DOM, so
it is unit-tested like the library; `web/main.ts` is the thin layer that renders that model. Vite is
pinned to **5** so the CI matrix can keep Node 18.

## Testing

Twenty-three tests. The library: the LCS diff (equal/insert/delete, changed lines, empty inputs,
stats), the comparison (file order, per-agent diff or null, overlap, duplicate-agent rejection) and the
renderer (self-contained output, escaping). The app: the DOM-free view model (summary totals, file-row
mapping, divergence flag, the path and "only diverged" filters). CI on Linux and Windows across Node
18/20/22 runs the library typecheck, tests and build plus the app's `typecheck:web` and `build:web`.

## Links

- Repository: https://github.com/BoogeyCodes/difflens
- Related: [runtrace.md](runtrace.md), [agentscore.md](agentscore.md)
