# runtrace

> Normalise heterogeneous AI coding-agent run transcripts into one schema and summarise them with
> metrics.

- **Repository:** https://github.com/BoogeyCodes/runtrace
- **Status:** Complete
- **Primary stack:** TypeScript (standard library only), tsup, Vitest
- **Last updated:** 2026-11-08

## Overview

runtrace coerces an agent's run log — whatever shape it arrived in — into a common `Trace` of typed
steps (command, file edit, file read, message, test), then reports the metrics an evaluator reaches
for first: commands run, files edited, lines changed, whether tests ran and passed. Zero dependencies,
ESM and CJS.

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

## Testing

Fourteen tests across field mapping, kind inference, bare strings, alternative container keys, line
counts, the metrics (dedup, pass/fail/unknown, failed commands) and the one-line summary. CI on Linux
and Windows across Node 18/20/22.

## Links

- Repository: https://github.com/BoogeyCodes/runtrace
- Related: [agentrubric.md](agentrubric.md), [difflens.md](difflens.md)
