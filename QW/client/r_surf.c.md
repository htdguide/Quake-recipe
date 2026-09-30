# QW/client/r_surf.c

> The surface cache's generator: a lit, mipped copy of a world surface built on demand.

**Needs** — as [`r_surf.c`](../../WinQuake/r_surf.c.md)
**Used by** — as [`r_surf.c`](../../WinQuake/r_surf.c.md)
**Tier floor** — as [`r_surf.c`](../../WinQuake/r_surf.c.md)

## Purpose

Read [`r_surf.c`](../../WinQuake/r_surf.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`r_surf.c`](../../WinQuake/r_surf.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Nothing of substance. The invalidation test — light style levels, the dynamic light set, the texture animation frame — is unchanged and remains the load-bearing content.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
