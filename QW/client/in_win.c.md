# QW/client/in_win.c

> The Windows input backend.

**Needs** — as [`in_win.c`](../../WinQuake/in_win.c.md)
**Used by** — as [`in_win.c`](../../WinQuake/in_win.c.md)
**Tier floor** — as [`in_win.c`](../../WinQuake/in_win.c.md)

## Purpose

Read [`in_win.c`](../../WinQuake/in_win.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`in_win.c`](../../WinQuake/in_win.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Nothing of substance. The accumulate-then-consume rule, the pitch clamp and the raw-device preference are unchanged — and they matter more here, because a command is now the unit of simulation ([`cl_input.c`](cl_input.c.md)).

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
