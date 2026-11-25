# sketchkit

> Dependency-free probabilistic data structures in Python (Bloom, Count-Min, HyperLogLog) — plus
> Sketchkit Explorer, a web app that measures each one against exact ground truth.

- **Repository:** https://github.com/BoogeyCodes/sketchkit
- **Status:** Complete (v0.2 — added the explorer app)
- **Primary stack:** Python 3.10+ (library: standard library only); FastAPI for the `[web]` extra
- **Last updated:** 2026-11-25

## Overview

sketchkit implements three classic probabilistic data structures. Each answers a question about a
stream of items using a fixed, small amount of memory, by allowing a bounded error: a Bloom filter
answers "have I seen this?", a Count-Min Sketch answers "how many times?", and a HyperLogLog answers
"how many distinct?". They are implemented from first principles in a form clear enough to read, and
the test suite checks that the statistical guarantees actually hold on real data.

v0.2 adds **Sketchkit Explorer** (`sketchkit serve`, the optional `[web]` extra), a FastAPI app that
makes the accuracy/memory trade-off tangible. Each demo runs the real library class against *exact*
ground truth computed server-side and shows the numbers side by side: the Bloom filter's measured
false-positive rate versus its theoretical one (and zero false negatives), the Count-Min sketch's
estimate versus the true frequency for the busiest items of a seeded Zipf stream (never an
underestimate), and the HyperLogLog's estimated versus true distinct count — each with the memory it
uses against an exact set or dict. Every demo has a JSON endpoint sharing one seeded compute function,
so the page and the API agree and are reproducible.

## Purpose and goals

Exact answers to these three questions need memory that grows with the data — a set, a counter
dictionary, another set — which is the whole problem at scale. These structures give approximate
answers in constant memory, and they are an elegant piece of computer science that rewards a careful
implementation. The goal was to build all three correctly, share the hashing properly, and prove the
guarantees with tests rather than just assert them in a comment.

Goals:

- Correct implementations of the three structures, sized from their error parameters.
- Reproducible hashing, so results do not change between runs.
- Tests that verify the actual guarantees (no false negatives, no underestimate, bounded error).

Non-goals: merging/union operations, persistence and a scalable (auto-growing) Bloom filter are
listed as future work.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | Python 3.10+ | Clear, and `int.bit_length` / `int.bit_count` make the bit work tidy |
| Hashing | `hashlib.blake2b` | A fast, process-stable digest (unlike the salted built-in `hash`) |
| Library dependencies | None | Nothing to install or audit |
| Explorer (`[web]` extra) | FastAPI + Uvicorn + Jinja2 | Demos and a JSON API over the real structures |
| Tests | pytest | Statistical guarantee checks, the demo compute, and the app |
| CI | GitHub Actions matrix | Python 3.10–3.13 |

## Architecture

```
  _hashing.py
    blake2b(item) -> 64-bit digest -> (h1, h2)   two 32-bit halves
    hashes(item, k, m): h1 + i*h2 mod m          k hashes from one digest
        │
        ├──► bloom.py        k bits in a bit array; "present" = all k set
        ├──► countmin.py     depth x width counters; estimate = min across rows
        └──► hyperloglog.py  2**p registers of leading-zero ranks; harmonic-mean estimate
```

All three share one hashing module, so each item is hashed once and the structures stay consistent
and reproducible.

The explorer sits on top without touching the structures: `demos.py` holds pure, seeded compute
functions that build a structure, run it against exact ground truth, and return a plain dict of
numbers (estimate, truth, error, memory, saving); `web.py` renders those as HTML and the same shape of
JSON. Because the compute is pure and seeded, it is unit-tested directly — the app's correctness is
checked the same way the library's is.

## Key design decisions

**One hash, many derived hashes.** The structures need several hash functions, but hashing an item
`k` times is wasteful. sketchkit hashes once with BLAKE2b, splits the 64-bit digest into two 32-bit
values, and derives every hash it needs as `h1 + i * h2` (the Kirsch–Mitzenmacher technique). This is
both faster and the standard approach for these structures.

**BLAKE2b instead of the built-in `hash`.** Python salts `hash` per process for security, so it
returns different values across runs. For data structures whose correctness depends on hashing the
same item to the same place every time, that is unacceptable; a cryptographic digest is stable across
processes and machines. The tests assert this reproducibility directly.

**Size from error parameters, not magic numbers.** The Bloom filter derives its optimal bit-count and
hash-count from the requested capacity and false-positive rate; the Count-Min Sketch has a
`from_error(epsilon, delta)` constructor that sets the grid dimensions from the formal guarantee; the
HyperLogLog's error follows from its precision. The APIs are phrased in the terms a user actually
cares about.

**One-sided errors, by construction.** A Bloom filter never gives a false negative (adding only sets
bits), and a Count-Min Sketch never underestimates (collisions only raise counters, and the estimate
takes the minimum). These are properties of the design, and the tests check that they hold in
practice.

## Data and interface

```python
BloomFilter(capacity, error_rate=0.01)
    .add(item) · item in bf · .update(iterable) · len(bf) · .bits_set · .estimated_error_rate()

CountMinSketch(width=2000, depth=5)  /  CountMinSketch.from_error(epsilon, delta)
    .add(item, count=1) · .estimate(item) · cms[item] · .total

HyperLogLog(precision=14)
    .add(item) · .count() · len(hll)
```

## Testing

Thirty tests, and the notable ones check the guarantees rather than just the plumbing:

- **Bloom:** every added item is found (no false negatives), and the observed false-positive rate on
  5,000 absent items stays near the configured 1%. Sizing matches the known optimum (~9.6 bits/item,
  7 hashes).
- **Count-Min:** across a skewed 20,000-item stream the estimate is never below the truth, and on a
  50,000-item stream the overestimate stays within `epsilon x total` for essentially all keys.
- **HyperLogLog:** estimates for cardinalities from 100 to 100,000 land within a few standard errors
  of the truth, duplicates do not inflate the count, and higher precision is measurably more accurate.

The explorer adds its own tests (47 in total): the seeded demo compute — Bloom gives no false
negatives and a believable measured FP rate, Count-Min never underestimates the sampled items, and the
HyperLogLog estimate lands within a few percent of the truth — and the FastAPI app via `TestClient`
(each demo page and JSON endpoint). CI runs the suite on Python 3.10 through 3.13.

## Running and deployment

`pip install -e ".[dev]"` then `pytest` for the library. The explorer is `pip install
"sketchkit[web]"` then `sketchkit serve`, or the included Docker image.

## Challenges and lessons

**A bit-rank bug in HyperLogLog.** The first implementation counted leading zeros against the wrong
bit-width — scanning a full 64-bit value as if it were the shorter suffix — which produced negative
ranks and estimates near zero. The statistical tests failed immediately and unmistakably, which is
exactly why they are worth writing: a HyperLogLog that is subtly wrong still returns plausible-looking
numbers, and only a guarantee check catches it. The fix was to take the low `64 - precision` bits as
the suffix and rank within that width.

**Testing randomness without flakiness.** Probabilistic guarantees are about expectations, so tests
use large inputs and tolerance bands (a few standard errors) rather than exact equalities, and rely
on the reproducible hashing so a run does not pass or fail by luck.

## Future work

- Union and intersection of Bloom filters and HyperLogLogs.
- A scalable Bloom filter that grows instead of being pre-sized.
- Serialisation so a sketch can be saved and merged across machines.

## Links

- Repository: https://github.com/BoogeyCodes/sketchkit
- Related docs: [flagpole.md](flagpole.md) and [ratekit.md](ratekit.md) (other hashing-based, deterministically tested work)
