# difflens

> Build side-by-side, multi-run diff views for comparing what several AI coding-agent runs changed.

- **Repository:** https://github.com/BoogeyCodes/difflens
- **Status:** Complete
- **Primary stack:** TypeScript (standard library only), tsup, Vitest
- **Last updated:** 2026-11-08

## Overview

difflens takes each agent's file changes (before/after contents), computes a line diff per file, lines
them up in a column per agent, flags the files more than one agent touched, and can render the whole
thing as a self-contained HTML page. Zero dependencies, ESM and CJS.

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

## Testing

Thirteen tests across the LCS diff (equal/insert/delete, changed lines, empty inputs, stats), the
comparison (file order, per-agent diff or null, overlap, duplicate-agent rejection) and the renderer
(self-contained output, escaping). CI on Linux and Windows across Node 18/20/22.

## Links

- Repository: https://github.com/BoogeyCodes/difflens
- Related: [runtrace.md](runtrace.md), [agentscore.md](agentscore.md)
