# qw-qc/models.qc

> Data: the declarations the model compiler reads to build the game's animated models — not part of the game logic.

**Needs** — nothing
**Used by** — the model-building tool; **not** listed in [`progs.src`](progs.src.md)
**Tier floor** — none

## Purpose

Written in the same language but compiled by a different tool. It declares, for each model, its source frames, its skin, its scale and
origin, and the names of its animation sequences — which the tool turns into the model files the game loads
([`modelgen.h`](../QW/client/modelgen.h.md) is the resulting format).

**It is not in the game's build list** ([`progs.src`](progs.src.md)), so none of it reaches the running game.

## State

Data read by an art tool; no run-time state.

## What it records

```text
per model:  the source art files and their order
            the skin image
            the scale and the origin offset the format stores
            the named frame ranges the game logic refers to
            the bounding box the compiler should record
```

**Invariants** —

- **The frame names here and the frame numbers in the game logic must agree.** The game logic refers to animation frames by number
  ([`player.qc`](player.qc.md)), and those numbers come from the order declared here. Reordering a model's frames silently breaks every
  animation that uses it.
- **The scale and origin are baked into the model file** and the renderer folds them into the transform
  ([`gl_rmain.c`](../QW/client/gl_rmain.c.md)), which is why model vertices are small integers.
- **The bounding box declared here is what the engine uses for collision against the model**
  ([`world.c`](../QW/server/world.c.md)), so it is a gameplay value, not an artistic one.

**Notes** — recorded because the recipe mirrors the tree and because the first invariant is a real hazard: **the model's frame ordering
is an unwritten contract between an art pipeline and the game logic.** A rebuild replacing the models must preserve the ordering or
regenerate the frame constants from it.
