# sandrun

> Run an AI coding agent's work against tests in an isolated sandbox, locally or in Docker.

- **Repository:** https://github.com/BoogeyCodes/sandrun
- **Status:** Complete
- **Primary stack:** Python 3.10+ — zero-dependency core; FastAPI + Jinja2 web layer (optional extra)
- **Last updated:** 2026-11-12

## Overview

sandrun materialises a task's files plus an agent's changed files into a throwaway workspace, runs a
test command under a timeout, and returns a structured pass/fail result — locally, or inside a Docker
container for stronger isolation. It is the objective-correctness half of agent evaluation: don't judge
by eye, run the code against the tests. It ships as three things in one package: a **web dashboard** (a
run history with a live pass rate), a **REST API**, and a **CLI + library**. The runner engine has no
dependencies; the web layer is an optional extra (`pip install sandrun[web]`).

## Purpose and goals

Deciding whether an agent actually fixed the bug means running its code against the tests, without
letting an unknown change touch your working tree or machine. sandrun is that harness, and it leans on
the exact skills the work needs — shell, subprocess, Docker and test pipelines.

## Key design decisions

**Files + overlay model.** `run_in_sandbox(files, command, overlay=...)` writes the starting repo, then
the agent's changed files on top, into a temp directory, runs the command there, and deletes it
afterwards — a clean, dependency-free way to represent "apply the agent's work, then run the tests".

**Timeouts and isolation.** Every run has a timeout and is killed if it overruns (reported as
`timed_out`); the workspace rejects path traversal; the Docker runner disables networking by default
and can cap memory. The inputs are untrusted code being evaluated, so this matters.

**Docker argv is testable without Docker.** `build_docker_argv` constructs the `docker run --rm`
invocation (bind-mount, workdir, `--network none`) as data, unit-tested on its own; `run_docker`
executes it when Docker is present.

## From a runner to a service

The runner answers "did this one command pass?" The service answers "what's the pass rate across all
the runs I've submitted?" — and lets a run be submitted over HTTP.

**A dependency-free SQLite history.** `RunStore` records each run (command, status, exit code, timing,
logs) using stdlib `sqlite3`, and `stats()` summarises them — passed/failed/timeout counts, pass rate,
average duration. A run's status (passed / failed / timeout) is derived from its `RunResult`, with an
`error` status reserved for a submission that could not even launch.

**A clean core/web boundary.** The web layer (`sandrun.service.create_app`, FastAPI + Jinja2) is an
optional extra; the runner never pulls in a web framework. Submitting a run (`POST /api/runs`, or the
dashboard form) materialises the files, executes in the sandbox — locally, or in Docker when an image
is named — and stores the verdict. The REST API exposes runs and `stats()`; each run gets a page with
its logs.

**Honest about the trust boundary.** The local runner executes submitted code on the host inside a
throwaway *directory*, not a container, so the docs and the dashboard say so plainly: bind to
localhost, and name a Docker image for untrusted code. Execution failures are caught and recorded as an
`error` run rather than crashing the request.

## Testing

Thirty-seven tests: the runner (pass, fail, timeout, cwd, empty command), the workspace/harness
(materialise/cleanup, overlay overwrite, path-traversal rejection, end-to-end agent fix), the Docker
argv builder, the SQLite store and stats, and every web route (dashboard, submit/delete, REST API,
stats). CI on Linux and Windows across Python 3.10–3.13.

## Links

- Repository: https://github.com/BoogeyCodes/sandrun
- Related: [patchkit.md](patchkit.md), [agentrubric.md](agentrubric.md)
