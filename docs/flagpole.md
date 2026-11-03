# flagpole

> A small feature-flag service with deterministic percentage rollouts, built on FastAPI.

- **Repository:** https://github.com/BoogeyCodes/flagpole
- **Status:** Complete
- **Primary stack:** Python 3.11+, FastAPI, Pydantic
- **Last updated:** 2026-11-03

## Overview

flagpole is a REST service for feature flags. A flag has a master on/off switch, per-user allow and
deny lists, and a percentage rollout. The interesting part is the rollout: asked "is this user in the
40%?", flagpole answers the same way every time for the same user, so a gradual rollout does not make
a feature flicker on and off between requests. It is a small, in-memory service with a clean API, so
the evaluation logic is the visible subject rather than the plumbing.

## Purpose and goals

Feature flags let a team deploy code switched off and turn it on gradually. The common failure is the
percentage rollout: if membership in "the 40%" is decided by a random number or a per-process hash, a
user moves in and out of the feature on every request or after every restart, which makes a rollout
impossible to reason about or debug.

Goals:

- Decide rollout membership **deterministically** per user, and make raising the percentage only ever
  add users (monotonicity), never remove them.
- Support explicit allow and deny lists alongside the percentage.
- Validate input at the edge so bad requests fail cleanly.
- Keep the evaluation logic pure and exhaustively testable.

Non-goals: persistence, multi-environment support and audit history are listed as future work. The
store is in memory behind a small interface so a database can be added without touching the API.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Framework | FastAPI | Typed request/response models and automatic OpenAPI with little code |
| Validation | Pydantic | Declarative model validation; bad input becomes a 422 for free |
| Server | uvicorn | Standard ASGI server |
| Hashing | `hashlib` (sha256) | A process-stable digest for rollout bucketing |
| Tests | pytest + FastAPI `TestClient` | Pure unit tests for logic, end-to-end tests for the API |
| CI | GitHub Actions matrix | Python 3.11, 3.12, 3.13 |

## Architecture

```
  HTTP request
      │
      ▼
  FastAPI routes (app.py)   validation via Pydantic models
      │ depends on
      ▼
  FlagStore (store.py)      in-memory dict, small interface
      │ flag
      ▼
  evaluate(flag, user)      (evaluation.py) — pure decision + stable bucket
      │
      ▼
  Evaluation { enabled, reason }
```

The store is injected through FastAPI's dependency system by `create_app(store)`, so each test gets a
fresh store and the routes never reach for global state.

## Key design decisions

**Determinism from a cryptographic hash.** The rollout bucket is `sha256(key + ":" + user) mod 100`.
Python's built-in `hash()` is salted per process, so using it would move a user to a different bucket
after every restart. A sha256 digest is stable across processes and machines, which is the exact
property a rollout needs. The test suite asserts determinism and that a user enabled at X% stays
enabled at every higher percentage.

**Rule precedence is explicit.** Evaluation applies rules in a fixed order — master switch, deny
list, allow list, then the percentage — and returns a human-readable reason for the decision. The
reason is part of the response, so an operator can see *why* a user got the answer they did.

**Evaluation is a pure function.** `evaluate(flag, user_id)` does no I/O, so its properties can be
tested cheaply and exhaustively, including a statistical check that a 30% rollout lands roughly 30% of
2,000 users on.

**In-memory behind an interface.** `FlagStore` is a dozen lines, but the API and the rules depend
only on its shape, so a PostgreSQL or Redis store would drop in without touching them.

## Data and interface

**Flag:** `key` (lowercase slug), `description`, `enabled`, `rollout_percentage` (0–100),
`enabled_for`, `disabled_for`.

**REST surface.**

```
GET    /health
GET    /api/flags                           list
POST   /api/flags                           create (201, 409 on duplicate)
GET    /api/flags/{key}                     fetch (404 if absent)
PUT    /api/flags/{key}                     replace
DELETE /api/flags/{key}                     delete (204)
GET    /api/flags/{key}/evaluate?user_id=   { enabled, reason }
GET    /docs                                Swagger UI
```

## Testing

Twenty tests. The evaluation logic is unit-tested in isolation for rule precedence, determinism,
monotonicity in the percentage, and the rough distribution of a partial rollout. The API is tested
end to end with FastAPI's `TestClient`: full CRUD, duplicate (409) and not-found (404) handling, key
and percentage validation (422), and the evaluate endpoint reflecting the lists and the rollout. CI
runs them across Python 3.11 to 3.13.

## Running and deployment

`pip install -e ".[dev]"` then `flagpole` (serves on port 8000, Swagger at `/docs`), or
`uvicorn flagpole.app:app --reload`. It is a service, so deployment would be a container behind a
process manager; persistence is the one addition a real deployment needs.

## Challenges and lessons

**The built-in hash trap.** The first instinct for bucketing is `hash((key, user)) % 100`. It passes
a quick manual test and fails silently in production, because Python salts `hash` per process, so
every restart reshuffles who is in the rollout. Writing the determinism test first surfaced it
immediately and forced the switch to `hashlib`. A good reminder that "works once" and "works
repeatably" are different claims, and only the test proves the second.

**Testing a percentage.** You cannot assert an exact count for a hash-based rollout, so the test
checks that 30% of 2,000 users lands in a sensible band and, more usefully, that membership is
monotonic in the percentage — a property that must hold exactly and catches real logic errors.

## Future work

- A persistent store (PostgreSQL or Redis) behind the same `FlagStore` interface.
- Audit history: who changed a flag and when.
- Per-environment flag values (dev / staging / prod).
- Bulk evaluation of all flags for a user in one request.

## Links

- Repository: https://github.com/BoogeyCodes/flagpole
- Related docs: [ratekit.md](ratekit.md) (the same emphasis on deterministic, testable logic)
