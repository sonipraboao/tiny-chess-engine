# tiny-chess-01 — chess engine in ≤4,096 bytes of Python

`main.py` is **4,069 bytes** as committed (cap 4,096): a one-line wrapper
(`import lzma,base64;exec(lzma.decompress(base64.b85decode(b'…')))`) that
unpacks and runs the engine source — compression with its unpacker is
explicitly allowed and counted. Python 3.13, standard library only; the
engine reads/imports/runs no other file. No model calls. Deterministic:
fixed node budgets, no clock, no unseeded randomness (Zobrist keys come
from a fixed-seed PRNG at startup).

## Protocol (brief + kit RULES.txt + practice prompt)

HTTP to the env at `$MAKEFUN_ENV_URL`: `POST /game {}` serves the game in
play (`status`, `fen`, `start_fen`, `moves`, `plies`); `POST /move
{"game": n, "move": "<uci>"}` plays a move, the house answers. Loop until
`status == "done"`. UCI moves, castling is the king's move; 10,000 ms per
move, 1,800 s per trial. Each turn the engine replays `start_fen` +
`moves` for already-occurred position keys (repetition scoring), then
searches the served FEN, node budget by game ply: 70k/45k/25k/12k.

## Engine (original code)

0x88 board; pseudo-legal movegen with make-then-test legality (en-passant
pins, castling through check, promotions). Negamax alpha-beta + PVS,
iterative deepening to the node cap, transposition table, quiescence
(captures + promotions, all evasions in check), check extension,
null-move pruning, late-move reductions, killer + MVV-LVA ordering,
game-history repetitions and the 100-ply rule scored 0 in search.
Evaluation: material 100/320/330/500/900 + piece-square tables for
pawn/knight/bishop and a middlegame king table, packed one char per
value (rooks/queens: material only — a size trade-off).

## Verification (local)

- Perft exact: startpos d3 8,902; Kiwipete d3 97,862; three more standard
  positions 43,238 / 9,467 / 1,486. Deterministic: same position → same
  move on repeated searches. Tactics: mate-in-1 and hanging-queen wins.
- Practice trials vs the real house gauntlet (kit env server, local):

| build | trial | score |
|---|---|---|
| v1 dev | trial-0 | 3.0/6 (W d2,d3,d4) |
| v2 dev | trial-0 | 4.0/6 (+ W nodes-100k) |
| v2 dev | trial-1 | 3.0/6 (W d2,d3,100k) |
| final main.py | trial-2 | 3.5/6 (W d2,d3,d4, draw 100k) |

  Losses are all to nodes-250k/500k. Practice Elo ≈ 1,280–1,450 (final
  ≈ 1,360), above the 1,154 qualification bar in every trial. Final
  build think time in trial-2: mean ≈ 2.2 s, p95 ≈ 3.6 s, max ≈ 4.9 s
  per move on a shared VM under competing load.
- Self-play A/B at 20k nodes: the final search design beat the previous
  one 5.5/7 in completed games. A match vs the public sunfish engine
  (far larger than the 4 KB house baseline) is logged in build-log.md.

## Assumptions

- The served FEN is authoritative; only transposition/killer tables are
  kept between moves of a game, cleared when a new game starts.
- Repetitions of game-history positions score 0 inside the search,
  mirroring the house engines; the env adjudicates the third occurrence.
- The engine never resigns (200-ply material adjudication dominates it).
