# QW/client/gl_rmain.c

> The hardware renderer's frame.

**Needs** — as [`gl_rmain.c`](../../WinQuake/gl_rmain.c.md)
**Used by** — as [`gl_rmain.c`](../../WinQuake/gl_rmain.c.md)
**Tier floor** — as [`gl_rmain.c`](../../WinQuake/gl_rmain.c.md)

## Purpose

Read [`gl_rmain.c`](../../WinQuake/gl_rmain.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`gl_rmain.c`](../../WinQuake/gl_rmain.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The view comes from the prediction ([`cl_pred.c`](cl_pred.c.md)) and the entity list from [`cl_ents.c`](cl_ents.c.md); the spectator camera decides whether the weapon model and the followed player's body are drawn ([`cl_cam.c`](cl_cam.c.md)). The depth-range partitioning, the quantized-normal lighting tables, the flat shadow and the mirror are unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
