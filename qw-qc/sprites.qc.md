# qw-qc/sprites.qc

> Data: the declarations the sprite compiler reads to build the game's sprites — not part of the game logic.

**Needs** — nothing
**Used by** — the sprite-building tool; **not** listed in [`progs.src`](progs.src.md)
**Tier floor** — none

## Purpose

As [`models.qc`](models.qc.md), for sprites: each sprite's source frames, its orientation type and its size, which the tool turns into
the sprite files the renderer draws ([`spritegn.h`](../QW/client/spritegn.h.md)).

## State

Data read by an art tool; no run-time state.

## What it records

```text
per sprite:  the source frames in order
             the orientation type -- facing the viewer, upright and facing,
               oriented by its own angles, or about a fixed axis
             the size and origin
```

**Invariants** — **the orientation type is chosen here and the renderer obeys it**
([`gl_rmain.c`](../QW/client/gl_rmain.c.md)), so whether a flame stays vertical is decided in this file, not in the renderer. That is the
right place for it and it is worth knowing where to look.

**Notes** — twenty-six lines. Recorded for completeness.
