# agentrubric

> Score, rank and report on AI coding-agent runs against a quality rubric.

- **Repository:** https://github.com/BoogeyCodes/agentrubric
- **Status:** Complete
- **Primary stack:** Python 3.10+ (standard library only)
- **Last updated:** 2026-11-08

## Overview

agentrubric is the scoring engine of an AI-agent evaluation workflow. Several agents are given the
same task; you rate each run on ten quality dimensions, and agentrubric computes a weighted overall
score (1–5), ranks the runs, and produces the written deliverable — a Markdown or JSON report with the
score matrix, the ranking and each run's rationale. It is a CLI and a library with no dependencies.

## Purpose and goals

Evaluating AI coding agents means judging which of several runs did a developer task best, rating each
on areas like instruction-following, correctness, testing, safety and honesty, scoring and ranking
them, and writing an evidence-based rationale. agentrubric standardises the scoring half: it turns a
set of per-dimension ratings into a consistent weighted overall, a defensible ranking (ties handled),
and a clean report, so every evaluation comes out in the same shape.

## Key design decisions

**A weighted ten-dimension rubric.** The default rubric is the ten quality areas with sensible
weights (correctness and instruction-following count most, efficiency and tool-use least). The overall
score is the weighted average on the 1–5 scale. A caller can supply their own `Rubric`.

**Standard competition ranking with tie flags.** Runs are sorted best-first; equal overalls share a
rank and are flagged `tied`, so a report never implies a false ordering.

**Validation at the edge.** An evaluation payload is checked against the rubric — every run must score
every dimension, with integers in 1–5, and agent names must be unique — raising a clear `EvaluationError`
rather than producing a misleading score.

**The report is the deliverable.** `to_markdown` renders the winner line, the dimension-by-agent
matrix, the overall/rank rows and the rationales; `to_json` gives the same as structured data.

## Testing

Fifteen tests over the rubric (weights, duplicate/zero-weight guards), scoring (weighted overall,
ranking, ties), validation (missing/unknown/out-of-range scores, duplicate agents) and the
Markdown/JSON reports and CLI. CI runs on Python 3.10–3.13.

## Links

- Repository: https://github.com/BoogeyCodes/agentrubric
- Related: [runtrace.md](runtrace.md), [difflens.md](difflens.md), [sandrun.md](sandrun.md)
