# QW/client/r_main.c

> The software renderer's frame, unchanged in substance.

**Needs** — as [`r_main.c`](../../WinQuake/r_main.c.md)
**Used by** — as [`r_main.c`](../../WinQuake/r_main.c.md)
**Tier floor** — as [`r_main.c`](../../WinQuake/r_main.c.md)

## Purpose

Read [`r_main.c`](../../WinQuake/r_main.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`r_main.c`](../../WinQuake/r_main.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The frame is built from the predicted view ([`cl_pred.c`](cl_pred.c.md)) rather than an interpolated received one, and the visible entity list is rebuilt each frame by [`cl_ents.c`](cl_ents.c.md) instead of being maintained. The renderer itself does not know the difference: it is handed a view and a list, which is why the whole span rasterizer survived the networking rewrite untouched. A few single-player-only debug commands are gone.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
