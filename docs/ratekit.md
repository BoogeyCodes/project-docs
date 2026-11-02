# ratekit

> Zero-dependency keyed rate limiting for TypeScript: token-bucket and sliding-window algorithms
> behind one small interface.

- **Repository:** https://github.com/BoogeyCodes/ratekit
- **Status:** Complete
- **Primary stack:** TypeScript (standard library only), built with tsup, tested with Vitest
- **Last updated:** 2026-11-02

## Overview

ratekit is a small library for limiting how often something may happen, per key. "Per key" is the
important part: you limit each user, IP address or API key independently, which is what a real
service needs. It offers the two algorithms that cover almost every in-process case — token bucket
and sliding window — behind a single `check(key)` method that returns a plain result, plus a helper
that turns that result into standard HTTP headers. It ships as both ESM and CommonJS with type
definitions and has no runtime dependencies.

It is deliberately small and framework-agnostic. There is no Redis, no Express coupling, no decorator
magic — just the limiting logic, with the algorithm visible and fully tested.

## Purpose and goals

Most rate-limiter packages sit at one of two extremes: a heavyweight that pulls in Redis and a web
framework, or a one-liner that hides the algorithm so you cannot reason about its behaviour at the
boundary. ratekit targets the common middle: an in-process limiter for a single service instance,
explicit about what it does.

Goals:

- Two well-understood algorithms with a shared, minimal interface.
- A result that tells the caller not just yes/no but exactly how long to wait, so a client can back
  off correctly.
- Bounded memory, because keys often come from an untrusted space (one per client IP).
- Deterministic tests, with no reliance on real elapsed time.
- Usable from any framework and from both ESM and CommonJS projects.

Non-goals: distributed limiting across instances (that needs a shared store like Redis and is listed
as future work), and async "wait until a token is free" scheduling — `check()` is synchronous and the
caller decides what to do with a denial.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | TypeScript 5 | Types are part of the contract for a library |
| Runtime deps | None | Nothing to audit or carry; the package is tiny |
| Build | tsup (esbuild) | One config emits ESM, CJS and `.d.ts` together |
| Tests | Vitest | Fast, TypeScript-native, good fake-value ergonomics |
| CI | GitHub Actions matrix | Linux and Windows, Node 18/20/22 |

## Architecture

```
  check(key, cost?)
       │
       ▼
  LruStore<State>   get/set by key; evicts least-recently-used past maxKeys
       │ per-key state
       ▼
  algorithm (token bucket | sliding window)
       │  reads time from an injected Clock
       ▼
  RateLimitResult { allowed, limit, remaining, retryAfterMs, resetMs }
       │
       ▼ optional
  rateLimitHeaders(result) -> RateLimit-* and Retry-After
```

Both limiters share two pieces: the `LruStore` that bounds memory, and the `Clock` that makes them
testable. Each limiter is a factory function returning a `Limiter` object (`check`, `reset`, `size`),
so there is no class hierarchy to learn.

## Key design decisions

**Two algorithms, one interface.** Token bucket and sliding window answer different questions —
"smooth the rate but tolerate bursts" versus "never exceed N in this window" — but both return the
same `RateLimitResult`. A caller can switch between them by changing the factory call and nothing
else.

**The result carries `retryAfterMs`, and it is honest.** When a request is denied, the result says
exactly how many milliseconds until a retry would succeed. The test suite asserts this directly:
advance the clock by `retryAfterMs` and the next check passes. That precision is what lets a client
back off correctly and what makes the `Retry-After` header trustworthy.

**Bounded memory via LRU.** Per-key state lives in an `LruStore` built on a plain `Map`. Because a
`Map` preserves insertion order, re-inserting a key on access moves it to the newest position and the
oldest key is always first, giving O(1) least-recently-used eviction with no dependency. This matters
because keys are often IPs or API keys from an unbounded, possibly hostile space; without a cap the
limiter itself becomes a memory-exhaustion vector.

**An injected clock instead of fake timers.** Time comes from a `Clock` interface. The synchronous
limiters never touch real timers, so tests advance a `ManualClock` rather than mocking the global
clock. This keeps the tests fast, obvious and portable.

**Nothing is consumed on denial.** A denied `check()` leaves the key's state untouched, so a rejected
request does not push the reset further out. This is the behaviour callers expect and is tested for
both algorithms.

## Data and interface

**Per-key state.** Token bucket stores `{ tokens, updatedAt }`; sliding window stores an array of hit
timestamps trimmed to the window.

**Public API.**

```ts
tokenBucket({ capacity, refillPerSecond, clock?, maxKeys? }): Limiter
slidingWindow({ limit, windowMs, clock?, maxKeys? }): Limiter
rateLimitHeaders(result): Record<string, string>
systemClock: Clock
new ManualClock(start?)        // advance(ms), set(ms)

interface Limiter {
  check(key: string, cost?: number): RateLimitResult
  reset(key: string): void
  readonly size: number
}

interface RateLimitResult {
  allowed: boolean
  limit: number
  remaining: number
  retryAfterMs: number
  resetMs: number
}
```

## Testing

Eighteen tests across the two limiters and the header helper, all driven by a `ManualClock`. They
cover: bursting up to capacity then denial; continuous refill at the configured rate; the idle bucket
never accruing beyond capacity; `retryAfterMs` being exactly sufficient; variable cost per check; key
isolation; LRU eviction past `maxKeys`; `reset`; the sliding window having no boundary spike; and
argument and option validation. CI runs the suite, a type-check and a build on Linux and Windows
across Node 18, 20 and 22.

## Running and deployment

`npm install` then `npm test`, `npm run typecheck` and `npm run build`. The build produces
`dist/index.js` (ESM), `dist/index.cjs` (CommonJS) and `dist/index.d.ts`, wired through the
package's `exports` map so both `import` and `require` resolve correctly. It is a library, so there is
no server to deploy; it would be consumed as a dependency (and is publishable to npm).

## Challenges and lessons

**Getting `retryAfterMs` right for the sliding window.** The naive answer — wait a full window — is
wrong. The correct wait is only until enough of the oldest hits age out to make room for the request,
which depends on the cost. Writing the test first ("advance by `retryAfterMs`, assert the next check
passes") forced the exact formula and caught an off-by-one in the index of the hit that must expire.

**LRU without a dependency.** It is tempting to reach for an LRU package, but JavaScript's `Map`
already iterates in insertion order, so delete-then-set on access and "evict the first key when
oversized" is a complete, O(1) LRU in a dozen lines. Keeping the dependency count at zero was worth
the small amount of code.

## Future work

- A Redis-backed store implementing the same `Limiter` interface, for limiting across instances.
- A fixed-window counter variant for callers who want the cheapest possible option.
- Ready-made middleware adapters for Express and Fastify in a separate entry point, to keep the core
  framework-free.

## Links

- Repository: https://github.com/BoogeyCodes/ratekit
- Related docs: [pulseboard.md](pulseboard.md) (where per-client rate limiting is a natural next step)
