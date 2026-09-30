# QW/server/profile.txt

> Data: a recorded execution profile of the dedicated server under load — the measured cost of every part of the server, in order.

**Needs** — nothing
**Used by** — the reader; and [`SYSTEM-REQUIREMENTS.md`](../../SYSTEM-REQUIREMENTS.md)'s conformance section
**Tier floor** — none

## Purpose

The recipe's only quantitative measurement of the shipped code, and therefore the best answer available to "where will my rebuild
spend its time, and how fast is fast enough". It is a real profile of a real server, not an estimate.

## State

Data; no run-time state.

## What it records

Each row gives a function's own time, its share of the total, its time including everything it calls, and **how many times it was
called**. The call counts are the more useful half.

**The shape of the server's cost, by inclusive share:**

| Share | What |
|---|---|
| ~21% | running player commands — the movement model and everything under it |
| ~9% | collision: the two hull sweeps, called about **6 million times** in the sample |
| ~9% | entity physics: thinking, pushing, tossing |
| ~6% | the game logic interpreter, about 108 thousand program executions |
| ~5% | building and writing the entity snapshots |
| ~4% | reading and sending packets |
| ~10% | loading files, once, at startup |
| ~6% | precomputing the hearable set, twice, at startup |

**Invariants and what they imply for a rebuild:**

- **Collision dominates.** The two recursive hull sweeps — the server's
  ([`world.c`](world.c.md)) and the player mover's ([`pmovetst.c`](../client/pmovetst.c.md)) — are together called about six
  million times in this sample and are the single largest steady cost. They are also short functions with no allocation. *This
  is the one routine in the whole engine worth optimizing first*, and it is the reason the hulls are precompiled and the plane
  tests are specialized for axis alignment.
- **Collecting the nearby world for the player mover is 6% by itself**, at 41 thousand calls — one per player command. That is
  the cost of [`pmove.h`](../client/pmove.h.md)'s flat-list design, paid per command rather than per frame, and it is why the
  list is capped at 32 entries.
- **The game logic interpreter is only about 6%** despite a hundred thousand program executions. A tree-walking or
  bytecode interpreter is fast enough for this workload, which is a useful licence for a rebuild: the game logic does not need
  compiling.
- **Linking entities into the world and finding which leaves they touch is 2–3%**, at over a million calls. That is the price of
  the spatial structure, and it is cheap.
- **Startup costs are visible and large** — file loading and the hearable-set precomputation together are about 16% of the
  sample — which is fine, because they happen once. But it confirms that the hearable set is not free
  ([`sv_init.c`](sv_init.c.md)) and that a map with many leaves will pause noticeably at load.
- **Reading the console costs over 1%**, which is the cost of the blocking-read loop that
  [`sys_unix.c`](sys_unix.c.md) replaces with a readiness wait. A measurable waste, and evidence for that change.
- Vector normalization and angle-basis construction appear at well over a hundred thousand calls each, which is why they are
  hand-written rather than general ([`mathlib.c`](../client/mathlib.c.md)).

**Notes** — a rebuild can use this as a target profile: if its own shape differs sharply — if the interpreter or the networking
dominates instead of collision — something is wrong structurally rather than locally. That is a far better conformance test than
a frame-rate number, and it is why this file is worth a page.
