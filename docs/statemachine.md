# statemachine

> A dependency-free declarative finite state machine for Python — plus Flowforge, a workflow engine
> and dashboard built on it.

- **Repository:** https://github.com/BoogeyCodes/statemachine
- **Status:** Complete (v0.2 — added the Flowforge workflow engine)
- **Primary stack:** Python 3.10+ (core: standard library only); FastAPI + SQLite for the `[web]` extra
- **Last updated:** 2026-11-18

## Overview

statemachine models a workflow as a set of states and the transitions between them. You declare the
transitions, then fire events with `trigger`; the machine moves state, running guards that can block a
move and callbacks that run on it, and refusing any move that is not allowed. It is a few lines to
learn and the core has no dependencies.

On top of it sits **Flowforge**, a workflow engine and dashboard. Where the library runs one machine
in memory, Flowforge treats workflows as *data*: a workflow is defined as JSON (states and
transitions, with a `requires` guard naming the data a move needs), stored, and instantiated. Each
running instance has a durable current state and an append-only history of every transition, so the
whole lifecycle is auditable. A dependency-free SVG renderer draws the machine with the current state
highlighted and the path taken marked, and a FastAPI app exposes it all as a dashboard and a REST API.

## Purpose and goals

Anything with distinct states and rules about moving between them — an order, a document, a
connection, a game — is a state machine, and expressing those rules as scattered `if` statements
quickly becomes unmanageable. The goal was to put the states, transitions, guards and side effects in
one declarative place, and to make invalid moves an error rather than a silent bug.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | Python 3.10+ | Dataclasses and callables keep it terse |
| Core dependencies | None | A small, self-contained library |
| Persistence | `sqlite3` (standard library) | Thread-safe store for workflows, instances and history |
| Diagram | Hand-written SVG | A layered layout with no Graphviz/Matplotlib dependency |
| Web (`[web]` extra) | FastAPI + Uvicorn + Jinja2 | Dashboard and REST API over the engine |
| Tests | pytest | Machine, definitions, diagram, store, engine, and the service via `TestClient` |
| CI | GitHub Actions matrix | ruff + pytest on Python 3.10–3.13 |

## Architecture

```
  WorkflowDefinition (JSON: states + transitions + requires guards)
        │ build_machine()                      diagram.render_svg()
        ▼                                            │
  StateMachine (the core library) ◀── replay at stored state   ▼ SVG (current + visited)
        ▲                                         WorkflowEngine
        │ fire()                                    define / start / fire / history
  WorkflowStore (SQLite: workflows, instances, transition history)
        ▲
        │
  FastAPI service (dashboard + REST API)  ──  statemachine serve / statemachine diagram
```

The engine is stateless: firing an event rebuilds the machine at the instance's stored state, applies
the transition, and persists the new state and a history row — so durability and the audit trail come
from the store, not from keeping a live object around.

## Key design decisions

**Declarative transitions with guards and callbacks.** `add_transition(trigger, source, dest)` where
source may be one state, a list, or `"*"` for any. A transition can carry guards (predicates that must
all pass) and an action; states carry `on_enter`/`on_exit`. Firing runs them in the order exit →
action → enter.

**Safe by default.** An invalid trigger raises `TransitionError` and leaves the state unchanged, so a
bad move is loud, not silent. `can(trigger)` and `triggers()` let callers check before acting.

**Event data threaded through.** Guards and callbacks receive an immutable `EventData` with the
trigger, source, destination, the machine, and any keyword arguments passed to `trigger`, so decisions
and side effects have the context they need.

## Testing

Forty-two tests. The core: basic transitions, invalid-move handling and state preservation, guards,
callback ordering, event data, wildcard and multi-source transitions. Flowforge: the declarative
definition layer and its validation; the SVG renderer (states, triggers, current-state highlight,
back-edges and self-loops); the SQLite store (CRUD, ordered history, stats); the engine (start, fire,
`requires` guards, history, invalid moves); and the FastAPI service via `TestClient` (creating
workflows, starting instances, firing actions with required fields, the JSON API, 404s). CI runs ruff
and the suite on Python 3.10–3.13.

## Challenges and lessons

**Deciding what a guard failure means.** A trigger can have several transitions; the machine takes the
first whose source matches and whose guards all pass, and if none qualifies it raises rather than
silently doing nothing. Making that explicit — and testing both the "guard blocks" and "no transition"
cases — kept the behaviour predictable.

## Future work

- Nested/hierarchical states, and timed/automatic transitions (deadlines).
- Per-transition webhooks so a workflow can drive external systems.
- A visual editor for building definitions in the browser.

## Links

- Repository: https://github.com/BoogeyCodes/statemachine
