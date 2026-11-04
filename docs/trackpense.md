# Trackpense

> A personal finance tracker built entirely in C# with Blazor: transactions, budgets and a live
> dashboard.

- **Repository:** https://github.com/BoogeyCodes/trackpense
- **Status:** Complete
- **Primary stack:** C# / .NET 8, Blazor Server, EF Core, SQLite
- **Last updated:** 2026-11-04

## Overview

Trackpense is a complete personal-finance app: record income and expenses, categorise them, set
monthly budgets, and see a dashboard of totals, spending by category, a six-month income-vs-expense
trend, and budget progress. It is built end to end in C# — an ASP.NET Core 8 Blazor Server front end
over an EF Core data layer — with the reporting logic in a separate, unit-tested library and the
charts drawn as inline SVG, so there is no JavaScript.

## Purpose and goals

The goal was a single, finished application that shows C# doing the whole job — data model, business
logic and interactive UI — rather than a backend paired with a separate JavaScript front end. It had
to be genuinely usable: real CRUD, real reporting, persistence, and a UI someone could actually run
their month through.

Non-goals: multi-user auth (it is single-user today), bank integrations and recurring transactions
are listed as future work.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| UI | Blazor Server (.NET 8, interactive) | An interactive SPA-like UI written in C#, server-rendered |
| Data | EF Core 8 + SQLite | The standard .NET model layer with zero setup |
| Charts | Inline SVG/HTML in Razor | No JavaScript charting dependency; renders server-side |
| Tests | xUnit + in-memory SQLite | The reporting logic tested directly |
| Ship | Multi-stage Dockerfile, GitHub Actions | One image; CI tests and smoke-tests the container |

## Architecture

```
  Trackpense (Blazor Server)
    Pages: Home (dashboard) · Transactions · Budgets
    Shared: MonthNav · BarChart · TrendChart (inline SVG)
        │ inject services
        ▼
  Trackpense.Core (plain library, no web dependency)
    Domain: Category · Transaction · Budget
    Data:   AppDbContext (EF Core/SQLite) · DbSeeder
    Services: TransactionService · BudgetService · SummaryService (+ MonthRef, DTOs)
        │ tested by
        ▼
  Trackpense.Tests (xUnit, in-memory SQLite)
```

## Key design decisions

**Domain and services in a web-free library.** All of the model and the reporting logic live in
`Trackpense.Core`, which knows nothing about Blazor. That is what lets the summary maths — monthly
income/expense/net, per-category spend against budget, the multi-month trend — be unit-tested in
isolation against an in-memory SQLite database, and keeps the UI a thin layer over it.

**A `DbContext` factory, not a scoped context.** Blazor Server keeps a single DI scope for the whole
circuit, so a scoped `DbContext` shared across concurrent component renders throws "a second
operation started on this context". Each service method instead creates a short-lived context from
`IDbContextFactory<AppDbContext>` — the pattern Microsoft recommends for Blazor — which makes the
services safe and, incidentally, easy to test with a factory over one in-memory connection.

**Charts without JavaScript.** The spending-by-category bars, the income/expense trend columns and
the budget progress bars are drawn in Razor from the data as plain HTML/SVG with CSS widths. The app
has no charting library and the charts render server-side like everything else, which keeps the whole
thing one language and one render model.

**Money as `decimal`, stored positive with a direction.** A transaction holds a positive `Amount` and
a `Kind` (`Income`/`Expense`); the signed value is derived. Totals and budget comparisons are exact
decimal arithmetic.

**Budgets are per category per month.** A unique `(CategoryId, Month)` index enforces one budget per
category per month, and setting a budget to zero removes it.

## Data and interface

**Entities.** `Category(Name, Color)`, `Transaction(Date, Amount, Kind, CategoryId, Note)`,
`Budget(CategoryId, Month, Limit)`.

**Pages.** Dashboard (totals, category chart, trend, budget bars), Transactions (add/edit/delete,
month filter), Budgets (set a monthly limit per category with progress). Month navigation is shared
across all three.

## Testing

Eighteen tests over the core library, each against an in-memory SQLite database:

- **SummaryService**: income/expense/net for a month, ignoring transactions outside it, grouping
  spend by category biggest-first, attaching budgets and flagging overspend (including the 112.5% →
  113% rounding), a category with a budget but no spend, and the per-month trend.
- **TransactionService**: add (trimming the note), month and category filtering, update, delete,
  missing-id handling, and validation (non-positive amount, missing category).
- **BudgetService**: create, update-in-place, clear on zero/null, and month scoping.

CI builds and tests on Linux and Windows, then builds the Docker image and smoke-tests the running
container.

## Running and deployment

`dotnet run --project src/Trackpense` for local use, or `docker build` + `docker run -p 8080:8080` for
the container. The database is created and seeded with a few months of demo data on first start.

## Challenges and lessons

**Blazor Server and `DbContext` lifetime.** The instinct is to inject `AppDbContext` directly, but
that breaks under Blazor Server's circuit-wide scope the moment two renders overlap. Switching the
services to an `IDbContextFactory` and a short-lived context per call fixed it and kept the services
testable. It is the detail that separates a demo that works in one click from one that works under
real use.

**Rounding a percentage.** Budget percent used `Math.Round`, whose banker's rounding turned 112.5%
into 112%. A budget indicator reads better rounding half up, so the model uses
`MidpointRounding.AwayFromZero` — a reminder that the default rounding is not always the one a user
expects.

## Future work

- Multiple accounts and transfers.
- CSV import/export, and recurring transactions.
- Authentication for a multi-user deployment.

## Links

- Repository: https://github.com/BoogeyCodes/trackpense
- Related docs: [pulseboard.md](pulseboard.md) (a .NET + React full-stack app, for contrast with this all-C# one)
EOF
echo "doc written"