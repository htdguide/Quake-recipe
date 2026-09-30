# WinQuake/gl_refrag.c

> Entity-to-leaf registration: which leaves of the world tree each visible entity touches, so the renderer can collect entities while walking the tree.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`gl_model.h`](gl_model.h.md)
**Used by** — [`cl_main.c`](cl_main.c.md) adds entities; [`gl_rsurf.c`](gl_rsurf.c.md) collects them while walking
**Tier floor** — none

## Purpose

Identical in substance to [`r_efrag.c`](r_efrag.c.md) — read that twin. An entity's bounding box is pushed down the world
tree and a small link record is made in every leaf it overlaps; walking the tree for visible surfaces then also yields
the entities that could be visible, with no separate visibility test per entity.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## What differs

Only the model header it compiles against ([`gl_model.h`](gl_model.h.md) rather than
[`model.h`](model.h.md)), which changes the bounding box's numeric type and nothing else.

**Notes** — the two files being separate is pure duplication with no design content. A rebuild has one copy; the recipe
mirrors the tree, so it has two twins and this one is a pointer.
