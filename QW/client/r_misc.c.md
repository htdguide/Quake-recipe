# QW/client/r_misc.c

> Renderer startup and per-map reset.

**Needs** — as [`r_misc.c`](../../WinQuake/r_misc.c.md)
**Used by** — as [`r_misc.c`](../../WinQuake/r_misc.c.md)
**Tier floor** — as [`r_misc.c`](../../WinQuake/r_misc.c.md)

## Purpose

Read [`r_misc.c`](../../WinQuake/r_misc.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`r_misc.c`](../../WinQuake/r_misc.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The per-map reset is now reached through a level change that does not disconnect ([`cl_main.c`](cl_main.c.md)), so it must be safe to call repeatedly on a live connection. The timing benchmark and the surface-cache report are unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
