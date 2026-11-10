# FairShare

> A shared-expense tracker (the Splitwise idea): groups, flexible expense splitting, running
> balances, and a minimal "settle-up" that collapses everyone's debts into the fewest payments.

- **Repository:** [github.com/BoogeyCodes/fairshare](https://github.com/BoogeyCodes/fairshare)
- **Status:** complete
- **Primary stack:** Python, FastAPI, SQLAlchemy, Jinja2 + HTMX
- **Last updated:** 2026-11-10

## Overview

FairShare is a complete little web app for tracking money shared between people — a trip, a flat, a
dinner. You create a group, add its members, record who paid for what and how to split it, and the app
keeps each person's running balance and tells you the fewest payments needed to settle up. It ships
both a JSON API and a server-rendered web UI, so it's usable from a browser and scriptable from code.

It's deliberately a *product*, not a library: authentication, a real domain with non-trivial money
maths, a web interface, and an end-to-end test suite.

## Purpose and goals

The goal was a finished Python application with genuine algorithmic depth rather than a thin CRUD
wrapper. Two pieces carry that weight: splitting an amount fairly in integer cents across four
strategies, and simplifying a web of debts into the minimum number of payments. A "done" version lets
a person register, build a group, log expenses with any split type, see correct balances, and get a
settle-up plan — all from a browser, with the same operations available over HTTP.

**Non-goals:** multi-currency, OAuth/third-party identity, and mobile apps. Session-cookie auth and a
single currency keep the focus on the domain.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Web framework | FastAPI | API + HTML from one app, dependency injection, OpenAPI for free |
| ORM | SQLAlchemy 2.0 + SQLite | Typed models, zero infrastructure to run |
| Templates | Jinja2 | Server-rendered pages |
| Interactivity | HTMX | The settle-up panel loads inline without a JS build step |
| Validation | Pydantic v2 | API request/response schemas |
| Auth | Starlette sessions + PBKDF2 | Signed session cookies; password hashing with the stdlib |
| Tests | pytest + Starlette TestClient | Unit tests for the core, HTTP tests for the app |

## Architecture

The money logic is pure and database-agnostic; a single service layer joins it to the ORM; the API and
web UI are thin layers over the services.

```
                 ┌───────────── pure core (no DB) ─────────────┐
HTTP ─▶ API ─┐   │  splitting.py   balances.py   money.py       │
             ├─▶ services.py ─▶ SQLAlchemy models ─▶ SQLite      │
HTTP ─▶ Web ─┘   └─────────────────────────────────────────────┘
                 services hold the rules; routes stay thin
```

- `splitting.py` turns an amount + a strategy into per-person cents.
- `balances.py` turns expenses + settlements into net balances, and net balances into a minimal set of
  transfers.
- `services.py` is the only module that imports both the ORM and the core; it enforces membership and
  authorization and persists results.

## Key design decisions

- **Money in integer cents, everywhere.** Floating-point dollars drift; cents don't. Every split
  returns integers that sum back to the exact total, with leftover pennies handed out by the
  largest-remainder method so the result is fair and deterministic.
- **Pure core, isolated from the database.** `splitting` and `balances` know nothing about SQLAlchemy,
  so the interesting logic is tested exhaustively with plain data and no fixtures. The ORM is a detail.
- **Greedy debt simplification.** Rather than record every pairwise IOU, FairShare nets each person to a
  single balance and matches biggest debtor to biggest creditor. This settles everyone in at most
  `n − 1` payments, and a person who nets to zero drops out of the plan entirely.
- **One app, two faces.** The same services back a JSON API and an HTMX web UI. Auth is a signed
  session cookie that works for both; a 401 on a browser route redirects to the login page, while the
  API keeps returning JSON.
- **Standard-library password hashing.** PBKDF2-HMAC-SHA256 from `hashlib` avoids a native crypto
  dependency and behaves identically across platforms and Python versions.

## Data and interface

**Entities:** `User`; `Group` (created_by); `Membership` (user ↔ group, unique); `Expense`
(group, description, amount_cents, paid_by, split_type); `ExpenseShare` (expense, user, share_cents,
summing to the expense total); `Settlement` (group, from_user, to_user, amount_cents).

**REST surface** (all group routes require membership):

| Method & path | Purpose |
| --- | --- |
| `POST /api/auth/register` · `/login` · `/logout` · `GET /api/auth/me` | Session auth |
| `POST/GET /api/groups` · `GET /api/groups/{id}` | Groups |
| `GET/POST /api/groups/{id}/members` | Members |
| `GET/POST /api/groups/{id}/expenses` | Expenses (equal / exact / percent / shares) |
| `GET /api/groups/{id}/balances` | Net balance per member |
| `GET /api/groups/{id}/settle` | Minimal settle-up transfers |
| `GET/POST /api/groups/{id}/settlements` | Recorded payments |

The web UI mirrors these at `/`, `/groups` and `/groups/{id}`.

## Testing

58 tests. The pure core is covered exhaustively: each split strategy (remainder distribution,
tie-breaking, validation errors), balance computation (nets to zero, settlements), and settle-up
(two-debtors-one-creditor, chain collapsing, the `n − 1` bound, already-settled). Money parsing and
formatting round-trip. The JSON API is driven over HTTP for auth, the 404-for-non-members rule,
owner-only actions, every split type and validation failures. A web-flow test registers, creates a
group, adds an expense through the form and reads the balances back off the page. Determinism comes
from a fresh in-memory SQLite database per test (shared via `StaticPool`).

## Running and deployment

```bash
pip install -e ".[dev]"
python -m fairshare          # http://127.0.0.1:8000  (web UI; API docs at /docs)
pytest                       # 58 tests
docker build -t fairshare . && docker run --rm -p 8000:8000 -v fairshare-data:/data fairshare
```

GitHub Actions runs ruff and the test suite on Python 3.11–3.13 and builds the Docker image.

## Challenges and lessons

- **In-memory SQLite across connections.** The first test run hit "no such table" because each
  connection to `sqlite://` gets its own database; the fix is a `StaticPool` so the whole app shares one
  connection. A good reminder that the pool, not just the URL, defines an in-memory database.
- **Required vs optional query params.** A non-nullable `bool` query parameter in FastAPI is *required*
  and 500s when absent — making booleans optional with a default avoids a surprising failure.
- **Keeping the core pure paid off.** Because splitting and settle-up never touched the ORM, the
  trickiest logic was the easiest to test and the quickest to get right.

## Future work

1. Multi-currency with per-expense currency and conversion at settle time.
2. An activity feed and per-expense edit history.
3. Email invites for people who don't have an account yet.
4. Export a group's ledger to CSV.
5. Optimistic HTMX updates for adding an expense (swap in the new balances without a full reload).
