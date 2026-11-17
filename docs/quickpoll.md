# quickpoll

> A full Ruby on Rails polling app: create a poll, share the link, and watch votes land live — with a
> clean JSON API underneath. One vote per person, kept honest by a signed cookie.

- **Repository:** https://github.com/BoogeyCodes/quickpoll
- **Status:** Complete (v0.2 — grew from an API into a full web app)
- **Primary stack:** Ruby on Rails 7.1 (full-stack), SQLite, Minitest
- **Last updated:** 2026-11-17

## Overview

QuickPoll is a live polling web application with a JSON API underneath. On the web you create a poll
(a question, two or more options, and an optional voting deadline), share its link, vote, and watch
the results update in place as votes arrive, with the leading option highlighted. A voter is
identified by a signed, long-lived cookie, so there is no sign-up and nobody can vote twice. The same
domain is exposed as a JSON API with a consistent error contract — and the two share one set of
controllers and one `results` method, diverging only at the edge via `respond_to`.

It began life as an API-only app; v0.2 flipped `config.api_only` off and added the web layer
(server-rendered views, cookie identity, CSRF, live results) without disturbing the API or its tests.

## Purpose and goals

A polls API is a compact domain with a surprisingly rich set of rules, which makes it a good, honest
demonstration of a Rails backend. The goal was to model it properly: real associations between polls,
options and votes; validations that enforce the rules that matter ("one vote per voter per poll", "an
option must belong to its poll", "no voting once a poll is closed"); a uniqueness guarantee backed at
both the model and the database level; and a consistent JSON contract for success and error.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Framework | Rails 7.1, full-stack | Server-rendered web UI plus the JSON API, one app |
| Views | Action View (ERB), inline CSS | A dark-themed UI with no asset pipeline to configure |
| Identity | Signed permanent cookie | One vote per browser, no sign-up; CSRF on for the web, skipped for JSON |
| Persistence | Active Record + SQLite | The standard Rails model layer; zero setup |
| Controllers | `ActionController::Base` + `respond_to` | One controller set serving HTML and JSON |
| Live results | JSON polling + progressive enhancement | Server renders correct results; a small script keeps them current |
| Tests | Minitest (Rails default) | Model, request (API) and integration (web) tests |
| CI | GitHub Actions + `ruby/setup-ruby` | Migrates a fresh database and runs the suite |

## Architecture

```
  routes.rb   root -> polls#index ; resources :polls (index/show/new/create, member close) + votes
        │
        ▼
  controllers (ActionController::Base)                 respond_to:
    ApplicationController                                html -> redirect / render view
      resolve_voter_token = params[:voter_token] || signed cookie id
      rescue_from RecordNotFound -> 404 page / JSON      json -> render / JSON error
      rescue_from RecordInvalid -> redirect+alert / 422
        │                                   │
        ▼                                   ▼
  views (ERB, inline CSS)               models
    index / new / show / _results        Poll has_many :options, :votes
    live results poll the .json view       #results, #accepting_votes?, #expired?, #leading_option
                                         Vote unique (poll_id, voter_token); option-belongs-to-poll;
                                           poll-open (closed or past expires_at)
```

## Key design decisions

**`poll_id` on the vote, so "one vote per poll" is a scoped uniqueness rule.** A vote references both
its option and its poll. Carrying `poll_id` directly lets the "one vote per voter per poll" rule be a
simple `uniqueness: { scope: :poll_id }` validation, and the same constraint is added as a unique
database index so a race cannot slip a duplicate past the validation.

**Validations express the domain rules.** Beyond presence and uniqueness, custom validations enforce
that a vote's option belongs to its poll and that a closed poll rejects new votes. Each rule is a
named validation with a clear message, and each has a test.

**A single JSON error contract.** `ApplicationController` rescues `RecordNotFound` into a `404` and
`RecordInvalid` into a `422` with the model's `full_messages`. Controllers stay thin — they call
`create!`/`find` and let the rescue handlers shape the error response — so success and failure are
consistent across endpoints.

**Results computed on the model.** `Poll#results` returns the serialised poll with each option's vote
count and percentage, keeping that logic in the model rather than scattered across controllers or
views. The same method feeds both the server-rendered results partial and the live-update JSON.

**One controller set for two formats.** Rather than a separate API namespace, each action derives the
voter identity once — an explicit `voter_token` if given (the API), otherwise a stable id from a
signed permanent cookie (the web) — then `respond_to` branches HTML versus JSON only for the
response. CSRF protection is enabled for the browser (forms carry a token) and skipped for JSON
clients; the test environment disables forgery protection as Rails conventionally does.

**Live results as progressive enhancement.** The poll page renders correct results server-side; a
small inline script polls the `.json` view and updates counts, percentages and bar widths in place.
With JavaScript off it still works — just without the live tick — so there is no hard dependency on a
realtime transport.

**Expiry as a model rule.** An optional `expires_at` gives a poll a deadline. `accepting_votes?`
(`!closed? && !expired?`) is the single predicate the vote validation and the views consult, so
"closed by hand" and "past its deadline" are handled uniformly.

## Data and interface

**Models.** `Poll(question, closed, expires_at)`, `Option(poll_id, label)`, `Vote(poll_id, option_id,
voter_token)` with a unique index on `(poll_id, voter_token)`.

**Web pages.** `/` (recent polls), `/polls/new` (create form with a dynamic option builder and a
deadline choice), `/polls/:id` (vote form or live results + share link + close control).

**Endpoints (JSON, via `Accept: application/json` or a `.json` suffix).**

```
GET    /polls              list, newest first
POST   /polls              create { question, options: [..], expires_at? }  (>= 2 options)
GET    /polls/:id          poll with results (adds accepting_votes, expires_at)
POST   /polls/:id/close    close to further voting
POST   /polls/:id/votes    cast a vote { option_id, voter_token }
```

A duplicate vote returns `422` with `{ "errors": ["Voter token has already voted in this poll"] }`.

## Testing

Forty-one tests. Model tests cover the vote rules (required token, one vote per poll, the same token
allowed in a different poll, an option from another poll rejected, closed and expired polls rejected),
the expiry predicates, and the results calculation. Request tests drive the JSON API end to end:
creating polls (and the `422`s for too few options or a blank question), listing newest-first, showing
results, closing, casting votes, the duplicate and closed-poll errors, and the computed percentages.
Integration tests drive the **web** flow: the home and new-poll pages render, creating a poll from the
form redirects and drops blank options, a browser session votes once (cookie-keyed) and then sees
results instead of the form, a second vote from the same session is refused, closing hides the form,
and an unknown poll renders the 404 page. CI migrates a fresh SQLite database and runs the whole
suite.

## Running and deployment

`bundle install`, `bin/rails db:prepare`, then `bin/rails server` (the web app on
`http://localhost:3000`) or `bin/rails test`. A Dockerfile builds a production image that migrates on
boot and serves with Puma; provide `SECRET_KEY_BASE` (it signs the voter cookie and session) and
mount a volume at `/app/storage` to persist votes. On Windows the bundled `tzinfo-data` gem supplies
the timezone database. `db/schema.rb` is generated by migration rather than committed, so CI migrates
a clean database per run.

## Challenges and lessons

**Writing Rails by hand, validated by CI.** This app was authored file by file rather than generated,
on a machine without Ruby, so the validation loop was GitHub Actions: push, let `ruby/setup-ruby`
install gems and migrate a fresh database, run the suite, and fix anything that failed. Keeping the
app api-only and the configuration minimal (requiring just the Active Record and Action Controller
railties) kept the boilerplate small and the surface for mistakes low — and the suite passed on the
first CI run.

## Future work

- Live results over Action Cable / Turbo Streams, replacing the JSON poll with a push.
- Rate limiting and abuse protection on voting (a natural fit for [ratekit](ratekit.md)).
- Richer poll management: an owner link to edit or reopen, and per-poll result exports.

## Links

- Repository: https://github.com/BoogeyCodes/quickpoll
- Related docs: [shortlink-api.md](shortlink-api.md) and [flagpole.md](flagpole.md) (other small, well-validated APIs)
