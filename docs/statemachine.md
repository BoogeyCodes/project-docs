# statemachine

> A small, dependency-free declarative finite state machine for Python: states, transitions, guards
> and callbacks.

- **Repository:** https://github.com/BoogeyCodes/statemachine
- **Status:** Complete
- **Primary stack:** Python 3.10+ (standard library only)
- **Last updated:** 2026-11-03

## Overview

statemachine models a workflow as a set of states and the transitions between them. You declare the
transitions, then fire events with `trigger`; the machine moves state, running guards that can block a
move and callbacks that run on it, and refusing any move that is not allowed. It is a few lines to
learn and has no dependencies.

## Purpose and goals

Anything with distinct states and rules about moving between them — an order, a document, a
connection, a game — is a state machine, and expressing those rules as scattered `if` statements
quickly becomes unmanageable. The goal was to put the states, transitions, guards and side effects in
one declarative place, and to make invalid moves an error rather than a silent bug.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | Python 3.10+ | Dataclasses and callables keep it terse |
| Dependencies | None | A small, self-contained library |
| Tests | pytest | Transitions, guards, callbacks, event data |
| CI | GitHub Actions matrix | Python 3.10–3.13 |

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

Eight tests covering basic transitions, invalid-move handling and state preservation, guards,
callback ordering, event data, and wildcard and multi-source transitions. CI runs on Python 3.10–3.13.

## Challenges and lessons

**Deciding what a guard failure means.** A trigger can have several transitions; the machine takes the
first whose source matches and whose guards all pass, and if none qualifies it raises rather than
silently doing nothing. Making that explicit — and testing both the "guard blocks" and "no transition"
cases — kept the behaviour predictable.

## Future work

- Nested/hierarchical states.
- A Graphviz export of the diagram.
- Async guards and callbacks.

## Links

- Repository: https://github.com/BoogeyCodes/statemachine
