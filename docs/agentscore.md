# AgentScore

> A web app for evaluating AI coding agents: rate three runs on a 10-dimension rubric, rank them, and
> export the review.

- **Repository:** https://github.com/BoogeyCodes/agentscore
- **Status:** Complete
- **Primary stack:** React 19, TypeScript, Vite, Vitest
- **Last updated:** 2026-11-08

## Overview

AgentScore is the front end of the agent-evaluation toolkit and a complete app in its own right. Three
agents are given the same task; AgentScore shows each run's summary, lets you rate it on the ten
quality dimensions with 1–5 pickers, computes a weighted overall (1–5) and a ranking live as you rate,
captures a rationale per run, and exports the written review as Markdown or JSON. It runs entirely in
the browser.

## Purpose and goals

Reviewing AI coding agents is a specific workflow — read each run, rate it on a rubric, score and rank
the runs, write a rationale. The goal was that workflow on a single screen, so every evaluation comes
out consistent and the deliverable is one click away.

## Key design decisions

**Scoring logic in a tested library.** The rubric, weighted overall, ranking and Markdown/JSON reports
live in `src/lib` with no React, so they are unit-tested directly; the UI is a thin layer over them.

**Live, editable ratings.** The ratings matrix (dimensions × agents) uses 1–5 pickers; the overall
scores, medals and ranks recompute on every change, with ties flagged.

**Export is the deliverable.** Copy the Markdown review to the clipboard, or download `.md`/`.json` —
the same report shape the CLI (`agentrubric`) produces.

**Opens on a realistic sample.** It starts with three agents' runs of "fix the failing login test" so
the workflow is visible immediately, then everything is editable.

## Testing

Eight tests: the scoring library (weighted overall, ranking, ties, the sample scoring) and the
Markdown/JSON reports, plus component tests that render the app and check the overall recomputes when a
score changes. CI type-checks, tests and builds.

## Links

- Repository: https://github.com/BoogeyCodes/agentscore
- Related: [agentrubric.md](agentrubric.md), [runtrace.md](runtrace.md), [difflens.md](difflens.md)
