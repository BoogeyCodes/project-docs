# Recall

> A spaced-repetition flashcards app built with Angular, driven by the SM-2 scheduling algorithm.

- **Repository:** https://github.com/BoogeyCodes/recall
- **Status:** Complete
- **Primary stack:** Angular 18 (standalone components, signals), TypeScript
- **Last updated:** 2026-11-03

## Overview

Recall is a flashcards app that schedules reviews with spaced repetition: you grade how well you
recalled each card, and the SM-2 algorithm decides when the card should come back. Cards you know well
return less and less often; ones you struggle with return soon. It is built with modern Angular —
standalone components, signals for state, and a service that persists the deck to localStorage — and
is kept small so the algorithm and the Angular idioms are the focus.

## Purpose and goals

Spaced repetition is what makes flashcard tools effective: reviews are timed so each card comes up as
you are about to forget it. The scheduling is a compact, elegant algorithm (SM-2), and the goal was to
implement it correctly as a pure, tested function and wire it into a clean Angular app that shows the
framework's current idioms.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Framework | Angular 18 | Standalone components and signals, the current idiom |
| State | Angular signals (`signal`, `computed`) | Reactive deck/due/stats without a store library |
| Persistence | localStorage | The deck survives a reload with no backend |
| Tests | Jasmine + Karma (headless Chrome) | Angular's default test stack |

## Architecture

```
  sm2.ts            pure SM-2 scheduler: (state, grade, now) -> next schedule
      │ used by
      ▼
  DeckService       cards = signal; due/stats = computed; review() runs SM-2 and
      │             reschedules; persisted to localStorage
      ▼
  components (standalone, OnPush, signal-driven)
    AppComponent     shell + statistics
    ReviewComponent  reveal the answer, then grade (Again/Hard/Good/Easy)
    AddCardComponent reactive form to add cards
```

Everything is reactive through signals: reviewing a card updates the deck signal, and the `due` and
`stats` computed signals, and the UI, follow automatically.

## Key design decisions

**SM-2 as a pure function.** The scheduler is a single pure function taking the card's state, the
grade and the current time, returning the next repetition count, interval, ease factor and due date.
No Angular, no I/O, so it can be unit-tested exhaustively: the `1 / 6 / interval × ease` progression,
the ease-factor adjustment per grade, lapse handling, and the 1.3 ease floor that stops a hard card
from spiralling into daily reviews.

**Signals over a store.** The deck is a `signal`, and `due` and `stats` are `computed` from it. For an
app this size, signals give the reactivity a Redux-style store would, with far less ceremony, and they
are the direction Angular itself is moving.

**Injectable, time-parameterised reviews.** `review(id, grade, now)` takes an optional clock, so the
service and scheduler can be tested deterministically rather than depending on the real date.

**Persistence that degrades gracefully.** The deck is saved to localStorage, wrapped in try/catch so
that in a private window or with storage blocked the app still runs; it just will not persist.

## Data and interface

A `Card` carries the SM-2 state (`repetitions`, `interval`, `easeFactor`), its `front`/`back`, a
`dueAt` timestamp and `lastReviewed`. The `DeckService` exposes `cards`, `due` and `stats` signals and
`addCard`, `review`, `remove` and `resetProgress`.

## Testing

Thirteen tests, run in headless Chrome. The SM-2 scheduler is tested in isolation for its progression,
ease adjustment, lapses, the ease floor, and that an easy card spaces out faster than a hard one. The
`DeckService` is tested for seeding, adding, reviewing (a card leaves the due set after a good grade,
stays after a lapse) and resetting. The shell component is tested for rendering. A `karma.conf.js`
with a no-sandbox launcher lets the same suite run on CI runners.

## Running and deployment

`npm install`, then `npm start` (dev server) or `npm run build`. It is a static SPA with no backend.

## Challenges and lessons

**Running Angular's tests on CI.** Angular's default Karma/Jasmine stack needs a real browser, and CI
runners need Chrome launched with `--no-sandbox`. Adding a small `karma.conf.js` with a
`ChromeHeadlessNoSandbox` launcher, plus a `test:ci` script, made the identical suite pass locally and
in CI. Worth knowing that "the tests pass" and "the tests pass in CI" are separate things for a
browser-based runner.

## Future work

- Multiple decks and tags.
- Import/export of cards as JSON or CSV.
- A review-history chart and a daily streak.

## Links

- Repository: https://github.com/BoogeyCodes/recall
