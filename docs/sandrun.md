# sandrun

> Run an AI coding agent's work against tests in an isolated sandbox, locally or in Docker.

- **Repository:** https://github.com/BoogeyCodes/sandrun
- **Status:** Complete
- **Primary stack:** Python 3.10+ (standard library only)
- **Last updated:** 2026-11-08

## Overview

sandrun materialises a task's files plus an agent's changed files into a throwaway workspace, runs a
test command under a timeout, and returns a structured pass/fail result — locally, or inside a Docker
container for stronger isolation. It is the objective-correctness half of agent evaluation: don't judge
by eye, run the code against the tests.

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

## Testing

Fifteen tests: the runner (pass, fail, timeout, cwd, empty command), the workspace/harness
(materialise/cleanup, overlay overwrite, path-traversal rejection, and an end-to-end "the agent's fix
makes the test pass"), the Docker argv builder, and the CLI. CI on Linux and Windows across Python
3.10–3.13.

## Links

- Repository: https://github.com/BoogeyCodes/sandrun
- Related: [patchkit.md](patchkit.md), [agentrubric.md](agentrubric.md)
