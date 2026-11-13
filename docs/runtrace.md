# runtrace

> Normalise heterogeneous AI coding-agent run transcripts into one schema and summarise them with
> metrics.

- **Repository:** https://github.com/BoogeyCodes/runtrace
- **Status:** Complete
- **Primary stack:** TypeScript — zero-dependency library (tsup, Vitest) + a Vite browser dashboard
- **Last updated:** 2026-11-13

## Overview

runtrace coerces an agent's run log — whatever shape it arrived in — into a common `Trace` of typed
steps (command, file edit, file read, message, test), then reports the metrics an evaluator reaches
for first: commands run, files edited, lines changed, whether tests ran and passed. Zero dependencies,
ESM and CJS.

On top of that library sits a **run-comparison dashboard** (Vite + vanilla TypeScript, client-side):
paste several transcripts in different shapes, each is normalised, and they are laid out side by side —
a per-run metrics table, each run's one-line summary, bars comparing commands/edits/lines, and a
"shared files" panel showing which runs touched the same files. The dashboard is a pure consumer of
the library's public API, so the engine and the product stay cleanly separated.

## Purpose and goals

Judging which agent did a task best starts with reading each run, but every agent/harness logs
differently. runtrace is the normalisation layer: map the many field names onto one schema, infer the
kind of each step when it isn't stated, and compute the comparison metrics, so the runs can be lined up
on the same terms.

## Key design decisions

**Tolerant field mapping.** Agent/task/final-message are read from several common key names; steps are
found under `steps`/`events`/`actions`/`trace`; a step's kind comes from `kind`/`type`/`tool`/`action`
with aliases (`bash`→command, `write`→file_edit).

**Inference when the kind is absent.** A `pytest`/`jest`/`go test` command becomes a `test` step, a
step carrying line counts becomes a `file_edit`, a shell-looking bare string becomes a `command`.

**Honest unknowns.** `testsPassed` is `true`/`false`/`null` — null when a test step reports no exit
code — so the metrics never imply certainty the log did not provide.

## The comparison dashboard

**A DOM-free comparison layer.** `web/compare.ts` turns several raw transcripts into a `RunComparison`
(per-run cards with metrics/summary/touched files, the cross-run shared files, and per-metric maxima
for scaling the bars) using only the library's public functions. Keeping it free of any DOM lets it be
unit-tested on its own, separate from the rendering. Runs that share an agent name get a `#n` suffix so
they don't merge.

**Static and dependency-free at runtime.** The dashboard is built with Vite and ships as static files
(no backend, no runtime dependencies), so it deploys anywhere — including the included nginx Docker
image. Vite 5 is pinned so the repo keeps Node 18 support across its CI matrix; the library continues
to build with tsup and publish as ESM + CJS.

## Testing

Twenty-two tests: the library's field mapping, kind inference, bare strings, alternative container
keys, line counts, the metrics (dedup, pass/fail/unknown, failed commands) and the one-line summary;
plus the comparison layer (card building, duplicate-label disambiguation, shared-file detection, and
bar maxima). CI on Linux and Windows across Node 18/20/22, and it builds both the library and the web
app.

## Links

- Repository: https://github.com/BoogeyCodes/runtrace
- Related: [agentrubric.md](agentrubric.md), [difflens.md](difflens.md)
