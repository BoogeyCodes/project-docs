# TaskForge

> A backend for a team project- and issue-tracker: projects, Kanban boards, issues with labels,
> assignees and comments, and real-time board updates over SignalR.

- **Repository:** [github.com/BoogeyCodes/taskforge](https://github.com/BoogeyCodes/taskforge)
- **Status:** complete
- **Primary stack:** C# / ASP.NET Core 8, EF Core, SignalR
- **Last updated:** 2026-11-09

## Overview

TaskForge is the API behind a small team's work board — the shape of a stripped-down Jira or Linear.
A user signs up, creates a project, invites teammates, and organises work as issues that move across
a Kanban board of columns. Issues carry a priority, an assignee, labels, a due date and a thread of
comments; the board can be filtered and paged; and because a board is something several people watch
at once, every change is pushed to connected clients in real time over SignalR.

It is built as a clean, test-covered ASP.NET Core 8 service: vertical-slice CQRS with MediatR, EF
Core over SQLite, JWT authentication, per-project authorization, RFC 7807 error responses, and an
integration suite that boots the real application and drives it over HTTP.

## Purpose and goals

The goal was a backend substantial enough to exercise the things a real product API needs, not a toy
CRUD: authentication and stateless authorization, a non-trivial domain with ordering and per-tenant
numbering, filtered and paginated queries, a real-time channel, and a test suite that proves the
whole stack works end to end.

A "done" version lets a client: register and log in; create a project and have a usable board appear;
invite members and have access enforced; create, edit, assign, label, comment on and **move** issues
with positions kept consistent; list issues with filters and paging; and receive live board events.

**Non-goals:** a UI (this is an API; it ships Swagger and an `.http` file), multi-region scale, and a
heavyweight identity provider — JWTs are issued directly, which is enough to demonstrate the auth and
authorization design.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Web | ASP.NET Core 8 Minimal APIs | Low-ceremony endpoints that map cleanly to use cases |
| Application | MediatR (CQRS) | One request + handler per use case; a single place for pipeline concerns |
| Validation | FluentValidation | Declarative rules run as a MediatR pipeline behavior |
| Persistence | EF Core 8 + SQLite | Rich model mapping with zero external infrastructure to run |
| Auth | JWT bearer + BCrypt | Stateless tokens; hashed passwords |
| Real-time | SignalR | Push board changes to subscribers of a project |
| Errors | RFC 7807 ProblemDetails | Consistent, machine-readable error contract |
| Logging | Serilog | Structured request logging |
| Docs | Swashbuckle / OpenAPI | Swagger UI with a bearer scheme |
| Tests | xUnit + FluentAssertions + WebApplicationFactory | Integration tests against the real app |

## Architecture

Vertical slices: each use case owns its request, validator, handler and DTOs. Endpoints are thin —
they translate HTTP into a MediatR message. Cross-cutting concerns are the only shared machinery.

```
HTTP ─▶ Minimal-API endpoint ─▶ MediatR
                                   │
                        ValidationBehavior (FluentValidation)
                                   │
                                 Handler ──▶ AppDbContext (EF Core / SQLite)
                                   │                │
                                   │         IProjectAccess (membership / ownership guard)
                                   │
                                   ├──▶ IBoardNotifier ──▶ SignalR BoardHub ──▶ subscribers
                                   ▼
                        result  ─or─  thrown exception
                                            │
                                   IExceptionHandler ──▶ RFC 7807 problem response
```

The composition root (`Program.cs`) wires EF, MediatR + the validation behavior, JWT bearer
authentication (including reading the token from the query string for SignalR connections),
authorization, SignalR, Serilog and Swagger.

## Key design decisions

- **Vertical slices over a service/repository stack.** A use case changes in one file. The shared
  pieces are deliberately few: validation, error mapping, paging and the access guard. The trade-off
  is some repetition across handlers, which I prefer to a web of shared services.
- **Authorization as data, not a role attribute.** A project should be invisible to people who aren't
  in it, so the check is "is there a membership row?" rather than a claim on the token. `IProjectAccess`
  centralises it and returns the *same* 404 whether a project is missing or simply not yours, so the
  API never reveals the existence of other teams' projects. Owner-only actions add a role check on top.
- **Per-project issue numbers.** Each project keeps a `NextIssueNumber`; issues read `WEB-12`, the
  identifier a human actually uses, instead of a surrogate GUID. The alternative — a global counter —
  leaks activity across projects and reads worse.
- **Explicit ordering with server-side reordering.** Columns and issues carry an integer `Order`.
  Moving an issue rebuilds the order of the affected column(s) so positions stay dense and stable,
  rather than trusting client-supplied indexes or fractional ranks.
- **DateTimeOffset on SQLite.** SQLite can't `ORDER BY` a `DateTimeOffset`. Rather than scatter
  client-side sorts, a value converter stores every (UTC) timestamp as sortable ticks, so ordering and
  paging stay in SQL. This was the one real impedance mismatch with the database.
- **Real-time gated by the same rule as REST.** The hub authenticates with the same JWT and runs the
  membership check before a connection may subscribe to a board — no second, weaker authorization path.

## Data and interface

**Entities:** `User`; `Project` (owner, unique key, next-issue-number); `ProjectMember` (user + role:
Owner/Member); `BoardColumn` (name, order, optional WIP limit); `Label` (name, colour); `Issue`
(per-project number, title, description, priority, assignee, due date, column, order); `IssueLabel`
(issue↔label join); `Comment` (author, body). Projects cascade to their board, issues and members.

**REST surface** (all project routes require a bearer token and project membership):

| Method & path | Purpose |
| --- | --- |
| `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/me` | Auth |
| `POST/GET /api/projects`, `GET/DELETE /api/projects/{id}` | Projects |
| `GET/POST /api/projects/{id}/members`, `DELETE …/members/{userId}` | Members |
| `GET/POST /api/projects/{id}/columns`, `PUT/DELETE …/columns/{columnId}` | Columns |
| `GET/POST /api/projects/{id}/labels`, `DELETE …/labels/{labelId}` | Labels |
| `GET/POST /api/projects/{id}/issues` | List (filter + page) / create |
| `GET/PUT/DELETE …/issues/{number}`, `POST …/issues/{number}/move` | One issue |
| `GET/POST …/issues/{number}/comments` | Comments |

**Events** (SignalR hub at `/hubs/board`, after `SubscribeToProject`): `IssueCreated`,
`IssueUpdated`, `IssueMoved`, `IssueDeleted`, each carrying the project, issue id, number and a summary.

## Testing

35 integration tests run against the real application via `WebApplicationFactory`, each class on its
own throwaway SQLite database with a controllable clock. They cover: registration/login and the JWT
path (including 401 when unauthenticated); project creation with the default board and owner; member
invitation, listing and removal with role enforcement; column and label lifecycles including the
"can't delete a column with issues" and "unique label" rules; the full issue lifecycle — sequential
numbering, default placement, filtering, pagination, the assignee-must-be-a-member rule, label sync,
and a move that reorders both the source and target columns; comments; validation (400) and
authorization (403/404) paths. A dedicated SignalR test connects a hub client with a JWT, subscribes
to a board, creates an issue over HTTP and asserts the `IssueCreated` event arrives.

Determinism comes from a fake clock and isolated databases; the SignalR test runs over long-polling
against the in-memory test server and waits on the event with a bounded timeout.

## Running and deployment

```bash
dotnet run --project src/TaskForge.Api      # http://localhost:5100/swagger
dotnet test                                 # 35 integration tests
docker build -t taskforge . && docker run --rm -p 8080:8080 -v taskforge-data:/data taskforge
```

The database is created on first run. The signing key is a development default overridden in
production via `Jwt__Key`. GitHub Actions restores, builds and tests on every push and also builds
the Docker image.

## Challenges and lessons

- **SQLite and DateTimeOffset.** The first run of the list endpoints threw at query time — SQLite
  rejects `ORDER BY` on a `DateTimeOffset`. A model-wide value converter to UTC ticks fixed it
  cleanly and kept sorting in the database. A good reminder to push timestamp handling into the model
  rather than working around it per query.
- **Testing SignalR in-process.** Driving a real hub client against `WebApplicationFactory` needs the
  test server's handler and the long-polling transport, and the JWT has to arrive via the query
  string because the browser client can't set an `Authorization` header — which is also why the
  production auth reads `access_token` for `/hubs` paths.
- **Keeping authorization in one place.** Routing every project-scoped handler through a single access
  guard kept the "invisible to non-members" rule consistent instead of re-deriving it per endpoint.

## Future work

1. Activity feed / audit log per project, reusing the board-event stream.
2. Saved board filters and swimlanes (group by assignee or label).
3. Column WIP-limit enforcement on move (currently advisory).
4. Refresh tokens and token revocation.
5. Move from `EnsureCreated` to EF migrations for schema evolution.
