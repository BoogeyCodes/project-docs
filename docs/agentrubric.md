# agentrubric

> Score, rank and report on AI coding-agent runs against a quality rubric.

- **Repository:** https://github.com/BoogeyCodes/agentrubric
- **Status:** Complete
- **Primary stack:** Python 3.10+ — zero-dependency core; FastAPI + Jinja2 web layer (optional extra)
- **Last updated:** 2026-11-12

## Overview

agentrubric is a platform for evaluating AI coding agents. Several agents are given the same task; you
rate each run on ten quality dimensions, and agentrubric computes a weighted overall score (1–5), ranks
the runs, produces the written report — and keeps a running **leaderboard** of how every agent does
across all of your evaluations. It ships as three things in one package: a **web dashboard**, a **REST
API**, and a **CLI + library**. The scoring engine has no dependencies; the web layer is an optional
extra (`pip install agentrubric[web]`).

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
matrix, the overall/rank rows and the rationales; `to_json` and `to_csv` give the same as structured
data.

## From a library to a platform

The scoring engine answers "which run won this one task?" The platform answers the question a reviewer
actually has: "which agent is better *over many tasks*?"

**A dependency-free SQLite store.** `EvaluationStore` persists each scored evaluation (using stdlib
`sqlite3`, so the core stays zero-dependency) and aggregates every agent across all of them into a
leaderboard — evaluations, wins, win-rate, average, best/worst. The rubric is serialised alongside each
evaluation, so a stored report still renders if the default rubric later changes.

**A clean core/web boundary.** The web layer (`agentrubric.service.create_app`, FastAPI + Jinja2) is an
optional extra; importing the scoring engine never pulls in a web framework. The dashboard offers a
submit form, per-evaluation report pages with Markdown/JSON/CSV export, and the leaderboard; the REST
API (`/api/evaluations`, `/api/leaderboard`, `/api/rubric`, `/healthz`) exposes the same for a harness.

**Thread-safety for the server.** The store shares one SQLite connection across FastAPI's threadpool
behind a lock (`check_same_thread=False`), and the app uses the modern lifespan API to open and close it.

## Testing

Thirty-four tests: the rubric (weights, duplicate/zero-weight guards), scoring (weighted overall,
ranking, ties), validation (missing/unknown/out-of-range scores, duplicate agents), the
Markdown/JSON/CSV reports and CLI, the SQLite store and leaderboard aggregation, and every web route
(dashboard, exports, REST API, submit/delete flows). CI runs on Python 3.10–3.13.

## Links

- Repository: https://github.com/BoogeyCodes/agentrubric
- Related: [runtrace.md](runtrace.md), [difflens.md](difflens.md), [sandrun.md](sandrun.md)
