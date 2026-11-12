# Gambit

> A chess engine you can play in the browser: a from-scratch, perft-verified move generator,
> alpha-beta search, and a click-to-play board — zero-dependency TypeScript.

- **Repository:** [github.com/BoogeyCodes/gambit](https://github.com/BoogeyCodes/gambit)
- **Status:** complete
- **Primary stack:** TypeScript, Vite, Vitest
- **Last updated:** 2026-11-12

## Overview

Gambit is a complete, playable chess program that runs entirely in the browser. It implements the
rules of chess from scratch — every legal move, including the awkward ones (castling through and into
check, en passant, under-promotion, pins) — searches for a reply with alpha-beta, and presents a
board you play by clicking. There is no chess library and no runtime dependency; the whole thing is a
few kilobytes of TypeScript.

## Purpose and goals

The goal was a project whose correctness is *provable*, not merely plausible. Chess move generation is
a classic "looks easy, is subtle" problem, and it has a gold-standard test — **perft**, which counts
the leaf nodes of the move tree to a given depth from known positions and compares against published
numbers. A "done" version matches those counts exactly, plays legal and sensible chess against a
human, and is playable in a browser with no setup.

**Non-goals:** a world-class engine (no transposition tables, quiescence search or opening book), and
networked/multiplayer play. The depth is in correctness and the search, not in rating.

## Tech stack

| Area | Technology | Why |
| --- | --- | --- |
| Language | TypeScript | One language for the engine and the UI, strictly typed |
| Build / dev | Vite | Instant dev server and a tiny static bundle |
| Tests | Vitest | Fast unit tests, including the perft suite |
| UI | Vanilla DOM | A board needs no framework; keeps the bundle dependency-free |
| Deploy | Static files / nginx (Docker) | It's a static site |

## Architecture

```
src/engine
├── board.ts     0..63 squares (a1 = 0), piece chars, FEN parse/format
├── attacks.ts   "is this square attacked?" — the basis of check detection
├── movegen.ts   pseudo-legal moves, make/unmake, legal filter (king safety)
├── perft.ts     node counting — the correctness oracle (+ perft-divide)
├── search.ts    material + piece-square evaluation, negamax + alpha-beta
└── notation.ts  UCI move strings
src/main.ts      the click-to-play board and game loop
```

**Legal moves** come from generating pseudo-legal moves, playing each with `makeMove`, checking
whether the mover's king is now attacked, and `unmakeMove`-ing. Because make/unmake mutates one board
in place rather than copying, perft to depth 4–5 runs in well under a second. **Search** is negamax
with alpha-beta and a capture-first move ordering over a material + piece-square-table evaluation.

## Key design decisions

- **Perft as the definition of done.** Rather than trust hand-written rule checks, correctness is
  pinned to published perft numbers: start to depth 4 (197,281), Kiwipete to depth 3 (97,862), and two
  tactical positions. Any missing en passant, illegal castle or bad pin would move these counts; they
  match to the node. This turned "is castling-through-check handled?" into a failing number, not a
  guess.
- **make/unmake over board copying.** Legality needs to try a move and look at the resulting position.
  Copying a 64-cell board per pseudo-move would dominate perft; instead each move records a small undo
  (captured piece and square, castling rights, en-passant square, clocks) and is reversed exactly.
- **Char board, not objects.** The board is a flat array of single characters (`'P'`, `'n'`, `'.'`).
  It's compact, cache-friendly and trivial to read in FEN order; piece colour is just letter case.
- **Castling legality in generation, not just the filter.** The legal filter only checks the king's
  final square, but a castle is illegal if the king *passes through* an attacked square — so that is
  checked where the castle move is generated.
- **No framework for the UI.** A chessboard is an 8×8 grid and a click handler; a framework would add
  more bytes than the engine. Event delegation on `data-square` attributes keeps it simple.

## Data and interface

The engine is a small pure API: `parseFen` / `toFen`, `legalMoves`, `makeMove` / `unmakeMove`,
`inCheck`, `isCheckmate` / `isStalemate`, `perft`, `bestMove`, and `moveToUci` / `moveFromUci`. The
UI holds a single `Position`, renders it, and drives the engine for the computer's side.

## Testing

13 tests. The heart is **perft** across four reference positions — the real proof the generator is
correct. Around it: rules tests for checkmate (fool's mate), stalemate, promotion (all four pieces),
en-passant capture and FEN round-tripping; and search tests that the engine finds a back-rank mate in
one and always returns a legal move. Determinism is free — the engine is pure and the positions are
fixed FENs.

## Running and deployment

```bash
npm install
npm run dev      # http://localhost:5173 — pick a colour and play
npm test         # 13 tests (perft + rules + search)
npm run build    # static bundle in dist/
docker build -t gambit . && docker run --rm -p 8080:80 gambit
```

GitHub Actions typechecks, tests and builds on Node 20 and 22, and builds the Docker image.

## Challenges and lessons

- **Perft earns its reputation.** The first run was off at depth 3; perft-divide (per-move node
  counts) pointed straight at the move whose subtree was wrong, which made debugging a matter of
  comparing one number rather than staring at rules. Having an exact oracle changed how I worked.
- **The fiddly rules are where bugs live.** En-passant capture square, castling-rights updates when a
  rook is *captured* on its home square, and promotion-while-capturing are each a line or two that are
  easy to get subtly wrong — and perft catches every one.

## Future work

1. Quiescence search and iterative deepening for stronger, steadier play.
2. A transposition table (Zobrist hashing) to cut the search tree.
3. SAN move display and a PGN export of the game.
4. A promotion picker (under-promotion) in the UI instead of auto-queen.
5. Run the search in a Web Worker so deep searches never touch the UI thread.
