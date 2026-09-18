# SaberTooth — Project Analysis & Roadmap

A candid assessment of where the engine stands today, what holds it back, and
what to build next. Companion to `README.md` (the "what") and `CLAUDE.md` (the
"how the code is organised"); this document is the "why and what next".

---

## 1. Current state

SaberTooth is a UCI chess engine in Rust (~7,350 lines) at version `0.2.0-alpha`.
It has outgrown the "toy engine" phase: the core is a correct, well-tested,
bitboard-based engine with a real search and a real evaluation. It is not yet
competitive — that is the natural next milestone.

What exists and works:

- **Board representation** — 12 bitboards (`piece + colour*6`), magic-bitboard
  sliding attacks, incremental Zobrist hashing, make/unmake with a `StateDelta`
  undo stack, FEN parsing, castling rights bitfield.
- **Move generation** — pseudo-legal generation filtered to legal via
  check/pin masks (O(1) bitboard tests); castling/en-passant fall back to
  make/unmake.
- **Search** — iterative deepening, fail-soft negamax, alpha-beta, aspiration
  windows, principal variation search, null-move pruning, late move reductions,
  check extensions, quiescence with delta + SEE pruning.
- **Move ordering** — TT move, promotions, MVV-LVA captures, killers, history
  heuristic.
- **Transposition table** — lock-free (Hyatt's XOR scheme), depth-preferred with
  generation aging, shared across Lazy-SMP threads.
- **Lazy SMP** — configurable `Threads` option; helpers diverge via order jitter
  and staggered start depth.
- **Evaluation** — PeSTO tapered material + PST, plus passed/isolated/doubled
  pawns, bishop pair, rook files, pawn shield, mobility, king danger.
- **Correctness oracle** — perft cases from the Chess Programming Wiki (start
  depth 5, Kiwipete depth 4, etc.) and a 300-position Win At Chess suite
  (~292/300 at depth 10).
- **Infrastructure** — CI (fmt, clippy with `pedantic`+`nursery`+`perf` denied,
  tests, perft-timing, security audit) and a release workflow for 4 targets.

---

## 2. Strengths

These are worth protecting as the engine grows.

1. **Correctness-first discipline.** Perft tests pin move generation to known
   counts, and the Zobrist hash is unit-tested to stay consistent through
   special moves, promotions, null moves, and unmake. This is the hard-won
   foundation everything else rests on.
2. **Incremental Zobrist hashing done properly.** XOR toggles for pieces,
   castling rights, en-passant file, and side-to-move; `unmake_move` restores
   the stored hash wholesale; tests compare against a full recompute.
3. **Modern move-generation legality.** Check/pin masks make most legality
   checks O(1) instead of per-move make/unmake, and double-check handling is
   explicit.
4. **Lock-free concurrency.** The TT avoids mutexes entirely, which keeps Lazy
   SMP simple and contention-free.
5. **Excellent documentation.** `#![warn(missing_docs)]`, rustdoc examples that
   double as tests, and clear module-level prose throughout.
6. **Strict linting.** Clippy `pedantic`/`nursery`/`perf` denied plus
   `warnings = deny` keeps the codebase clean and future-proof.

---

## 3. Weaknesses

Ordered roughly by impact. The first group costs the most Elo/nps; the later
groups are completeness and robustness.

### 3.1 Per-node allocation (biggest performance drag)

- `generate_moves` returns a `Vec<Move>` **every node**, and `order_moves`
  allocates a second `Vec<(i32, Move)>` for scoring. Allocation is the dominant
  cost in the search hot path.
- Move ordering scores **everything, then sorts**. Engines instead stage
  generation (TT move → captures → killers → quiets) and pick the best remaining
  move by scanning (selection sort) without materialising a score array.
- `get_piece_type_containing_position` / `get_piece_at` scan all 12 bitboards;
  they run in `order_moves`, `is_tactical`, `see`, and `commit_verified_move`.
  A `piece_on[64]` mailbox (or similar O(1) lookup) would remove a constant
  factor from every node.

### 3.2 Evaluation is recomputed from scratch

- `evaluate` rebuilds material, PST, mobility (magic attacks per piece), and
  king danger (attack maps per enemy slider) at **every leaf**. Mobility and
  king danger are the expensive parts.
- There is no incremental evaluation. An incremental material/PSST accumulator
  (updated in make/unmake) would cut leaf cost substantially.

### 3.3 Missing standard search pruning

Present: null-move, PVS, LMR, check extension. Absent:

- **Futility pruning** (skip hopeless quiet moves near the leaves) and
  **reverse futility pruning / static null move** (early cutoff on a large
  static margin). These are cheap and worth a lot in modern engines.
- **Internal iterative deepening** (probe TT for an ordering move when no TT
  move is found at high depth).
- **History-driven LMR** — history is used for ordering but not to size
  reductions; no continuation/quiet-history tables.

### 3.4 No tuning / measurement loop

The evaluation weights are explicitly hand-picked ("tuning waits for
self-play"). There is no self-play harness, no SPRT / Elo measurement, and no
`bench` command to detect perf regressions. Without this loop, every "improvement"
is a guess — the classic trap for engines at this stage.

### 3.5 Transposition table limitations

- **Single slot per index** (no 2–4 slot buckets), so an important entry can be
  evicted by a shallow one under pressure.
- **Fixed 64 MiB**, and no UCI `Hash` option to size it.
- Score stored as `i16`; safe for the current `MATE_SCORE` bound but fragile if
  the score scale ever widens.
- `clear()` zeroes all 4M slots on `ucinewgame` (generation aging already
  exists; clearing is redundant work).

### 3.6 UCI completeness

- Only `Threads` is exposed. Missing `Hash`, `MultiPV`, `Ponder`, `UCI_Chess960`.
- **No `bench` command** — the standard way to verify deterministic, measurable
  performance across changes.
- No pondering; time management is a flat `clock/25 + inc/2` with no "easy move"
  detection or move-count (`movestogo`) awareness (it is parsed but unused).
- No `bestmove … ponder` token.

### 3.7 Evaluation depth

- No connected-pawn, backward-pawn, or space terms; king safety is a coarse
  "attack unit" table with no pawn-storm/safety nuance; no bishop colour
  complex.
- `Piece::get_piece_value` is a second, unused material scale (pawns 200) that
  diverges from the real eval and only adds confusion.

### 3.8 Robustness / edge cases

- `halfmove_count` (`u8`) and `fullmove_count` (`u16`) can theoretically overflow
  in extreme games; the fifty-move check masks this in practice but the counters
  are not saturating.
- Repetition detection is twofold (documented trade-off, but it can mis-score in
  rare positions).
- A few leftover `TODO`/`refactor` markers in `moves.rs` and `move_generation.rs`.

### 3.9 Testing / tooling gaps

- No fuzz targets for the FEN parser or UCI parser (untrusted input).
- Coverage job is commented out in CI.
- `CLAUDE.md` references `compare.sh` / `run_stockfish.sh`, which no longer
  exist in the repo (doc drift).
- No benchmark/NPS reporting for search (only perft timing).

---

## 4. Next steps

Prioritised; early items are high-impact-per-effort, later items need the
earlier foundations in place.

### Phase 1 — Performance foundation (nps, then Elo)

1. **Kill per-node allocation.** Replace `Vec<Move>` with a fixed-size move list
   (e.g. `[Move; 256]` / a `MoveList` with a `len`) and stage move ordering
   instead of score-and-sort.
2. **O(1) square lookup.** Add a `piece_on: [u8; 64]` mailbox (or equivalent)
   updated in `add_piece`/`remove_piece`; use it in `order_moves`, `is_tactical`,
   `see`, and make/unmake.
3. **Incremental evaluation.** Accumulate material + PST (and phase) in
   make/unmake; keep mobility/king-danger as a fallback for the nodes that need
   it (or recompute lazily).
4. **Add a `bench` command** (a fixed set of positions searched to fixed depth)
   so every later change is measurable.

### Phase 2 — Search strength

5. **Futility pruning** and **reverse futility pruning** near the leaves.
6. **History-aware LMR** and continuation/quiet-history tables.
7. **Internal iterative deepening** for PV nodes.
8. Improve move ordering tiers (counter-move heuristic) as a cheap add-on.

### Phase 3 — Evaluation & tuning

9. Add incremental-eval-friendly terms (connected pawns, king safety
   refinements) and remove the dead `get_piece_value` scale.
10. **Build the self-play / SPRT harness** so eval weights and search changes
    are validated by Elo, not intuition. Consider open-source tuners (e.g.
    logistic regression on self-play games) for the eval.

### Phase 4 — Engine robustness & UCI

11. Expose `Hash` (and eventually `MultiPV`/`Ponder`); size the TT from the
    option and add multi-slot buckets.
12. Improve time management (easy-move detection, use `movestogo`).
13. Chess960 (castling rework) if compatibility matters.
14. Saturating move counters; address remaining `TODO` markers.

### Phase 5 — Testing & measurement

15. Fuzz the FEN and UCI parsers.
16. Re-enable the coverage job; wire a search NPS benchmark into CI alongside
    the existing perft timing.
17. Fix `CLAUDE.md`/`README.md` references to removed scripts.

---

## 5. Summary

SaberTooth's fundamentals are unusually solid for its stage — correct move
generation, a clean bitboard core, disciplined hashing, and a genuinely modern
search skeleton. The gap to "competitive" is now almost entirely about **hot-path
performance** (allocations, O(12) lookups, from-scratch eval) and **the missing
measurement loop** (no bench, no self-play, no tuning). Attack those in order and
each step is both measurable and additive.
