# ShortLink API

> A small, production-shaped URL shortener built with ASP.NET Core 8 minimal APIs.

- **Repository:** https://github.com/BoogeyCodes/shortlink-api
- **Status:** Complete
- **Primary stack:** ASP.NET Core 8, MediatR, FluentValidation, EF Core, Serilog
- **Last updated:** 2026-11-02

## Overview

ShortLink is a URL shortener: it takes a long URL, hands back a short code, and redirects visitors
from that code to the original. It supports custom codes, optional expiry, hit counting and per-link
statistics. The whole thing is a few hundred lines, but it is wired the way a real service is rather
than as a toy.

The project exists to answer, in a form someone can read quickly, the question "how do you structure
a .NET API?" Every piece is something I reach for on real work: CQRS slices, validation in the
pipeline, a persistence layer, structured errors, logging, integration tests against the real HTTP
stack, and a container build. It is deliberately small so each of those is visible rather than buried.

## Purpose and goals

The goal is a reference backend that is complete but not bloated. A URL shortener is a good vehicle
because the domain is understood in one sentence, which lets the structure be the subject. "Done"
means: create, redirect, inspect and delete links; reject bad input with clear errors; expire links
on schedule; count hits; and prove all of it with tests that exercise the real application.

Non-goals: no user accounts, no custom domains, no analytics dashboard. Those are listed as future
work so the scope stays honest.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Framework | ASP.NET Core 8 Minimal APIs | Thin HTTP surface with typed results and OpenAPI |
| Request handling | MediatR | One handler per use case, HTTP kept separate from logic |
| Validation | FluentValidation, run as a MediatR pipeline behaviour | Every request validated before its handler, in one place |
| Persistence | Entity Framework Core 8 (SQLite) | Simple model; provider swaps for a real database |
| Errors | `IExceptionHandler` mapping to RFC 7807 `ProblemDetails` | Consistent, documented error responses |
| Logging | Serilog with request logging | Structured, searchable logs |
| Docs | Swashbuckle / Swagger UI | Interactive API docs |
| Testing | xUnit + `WebApplicationFactory` | Boots the real app with a fake clock and a temp database |
| Ship | Multi-stage Dockerfile, GitHub Actions | Build, test and image in CI |

## Architecture

```
  HTTP request
     │
     ▼
  Minimal API endpoint  (Endpoints/LinkEndpoints.cs)   — HTTP only, no logic
     │ sends a MediatR request
     ▼
  ValidationBehavior  — runs every FluentValidation validator for the request
     │
     ▼
  Feature handler  (Features/Links/*.cs)   — one file per use case
     │   CreateLink · ResolveLink · GetLinkStats · DeleteLink
     ▼
  AppDbContext (EF Core)  ──►  SQLite

  Cross-cutting: ExceptionHandler maps domain exceptions to ProblemDetails;
  Serilog logs every request; the redirect lives at the root (/{code}).
```

The code is organised in vertical slices: each use case is a single file holding its request,
response, validator and handler. Adding a feature means adding a file, not editing five shared ones.

## Key design decisions

**Vertical slices over layered folders.** Rather than `Controllers`, `Services`, `Repositories`, each
feature is self-contained. For a small API this keeps related code together and makes each use case
easy to find and test.

**Validation as a pipeline behaviour.** Validators run automatically for any request before its
handler, so a handler never receives invalid input and no handler repeats validation code. It also
blocks dangerous input — only absolute `http`/`https` URLs are accepted, which keeps `javascript:`
and `data:` schemes out of the store.

**Short codes from a reduced alphabet.** Codes are seven characters from a 55-character alphabet that
omits look-alikes (`0/O`, `1/l/I`). That is about 1.5 × 10¹² combinations; collisions are still
checked against the database and retried, so correctness does not rely on luck.

**Expiry evaluated on read, not by a sweeper.** A link is checked for expiry when it is resolved, and
an expired link returns `410 Gone` rather than `404`, so clients can tell "never existed" from "used
to exist". This avoids a background job for a demo-scale service.

**An injected clock.** Time comes from an `IClock`, so expiry is tested by advancing a fake clock
instead of sleeping. Deterministic tests are worth the tiny abstraction.

## Data and interface

**Entity.** `Link` (id, code, target URL, created-at, optional expires-at, hit count, last-accessed).

**REST surface.**

```
POST   /api/links            create a link (generated or custom code, optional TTL)
GET    /{code}               302 redirect to the target, records the hit
GET    /api/links/{code}     metadata, hit count, last access, expiry state
DELETE /api/links/{code}     remove a link
GET    /health               liveness
GET    /swagger              interactive docs
```

The public redirect sits at the root so short URLs stay short: `https://host/{code}`.

## Testing

Eighteen tests. The integration tests boot the full application in-process with
`WebApplicationFactory` against a throwaway SQLite file, and replace the clock with a controllable
fake so expiry is tested by moving time forward. They cover create, validation rejection, redirect
and hit counting, expiry returning `410`, delete, conflict on duplicate custom codes, and that the
OpenAPI document generates without schema clashes. Unit tests cover the code generator's alphabet,
length and uniqueness. CI builds, tests and builds the Docker image on every push.

## Running and deployment

`dotnet run --project src/ShortLink.Api`, then open `/swagger`. A `shortlink.db` SQLite file is
created on first run. Or `docker build -t shortlink-api . && docker run -p 8080:8080 shortlink-api`.
A `requests.http` file is included for the VS Code REST Client.

## Challenges and lessons

**Swagger schema id clashes.** The vertical slices each nest a `Response` type
(`CreateLink.Response`, `GetLinkStats.Response`), and Swashbuckle's default schema id is just the
type name, so generation threw a 500 on a collision. The fix was a custom schema id that includes the
declaring type. A reminder that the convenient default breaks as soon as two nested types share a
name — and a reason the test suite now asserts the OpenAPI document generates.

**SQLite file locks in tests.** Deleting the temp database after a test failed because SQLite's
connection pool still held the file. Clearing the pool before deleting fixed it. Small, but the kind
of thing that makes a suite flaky if ignored.

## Future work

- API key authentication for the management endpoints.
- Rate limiting on link creation via the built-in rate limiter.
- A PostgreSQL provider with EF migrations for multi-instance deployment.
- QR code generation per link.

## Links

- Repository: https://github.com/BoogeyCodes/shortlink-api
- Related docs: [pulseboard.md](pulseboard.md) (the same patterns at larger scale)
