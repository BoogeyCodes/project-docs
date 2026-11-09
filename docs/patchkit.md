# patchkit

> Parse, summarise and apply the unified diffs AI coding agents produce.

- **Repository:** https://github.com/BoogeyCodes/patchkit
- **Status:** Complete
- **Primary stack:** Python 3.10+ (standard library only)
- **Last updated:** 2026-11-08

## Overview

patchkit reads a unified diff into a structured model, summarises what it touches (files, hunks, lines
added/removed), classifies each file (new/deleted/renamed/modified), and applies the patch to in-memory
file contents so the result can be checked — all without shelling out to `git apply`, and with no
dependencies.

## Purpose and goals

An AI coding agent hands its work back as a unified diff. To evaluate a run you need to know what the
diff actually does and whether it even applies cleanly to the given code. patchkit is the small toolkit
for that: a parser, a summariser and an in-memory apply.

## Key design decisions

**A faithful unified-diff parser.** Handles `diff --git`, `---`/`+++` headers, `@@` hunks and
`/dev/null` for new/deleted files, counting added/removed lines per hunk and per file.

**Apply with a precise failure.** `apply_patch` returns a new `path -> contents` mapping without
mutating the input; a hunk whose context does not match raises `ApplyError` naming the exact line, so
you learn immediately that a patch does not fit the code it claims to.

**Git-style summaries.** `summary()` gives the one-liner, `stat()` the per-file `(path, added, removed)`.

## Testing

Nineteen tests over parsing (modifications, new files, deletions, multi-file summaries, hunk counts,
singular/plural wording, non-diff rejection), applying (modify, create, delete, add-only hunks, context
mismatch, missing target, immutability) and the CLI. CI on Python 3.10–3.13.

## Links

- Repository: https://github.com/BoogeyCodes/patchkit
- Related: [sandrun.md](sandrun.md), [difflens.md](difflens.md)
