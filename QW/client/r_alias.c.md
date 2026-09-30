# QW/client/r_alias.c

> The animated-model rasterizer for the software renderer.

**Needs** — as [`r_alias.c`](../../WinQuake/r_alias.c.md)
**Used by** — as [`r_alias.c`](../../WinQuake/r_alias.c.md)
**Tier floor** — as [`r_alias.c`](../../WinQuake/r_alias.c.md)

## Purpose

Read [`r_alias.c`](../../WinQuake/r_alias.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`r_alias.c`](../../WinQuake/r_alias.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The per-player colour translation is applied from the table the client builds when a player's colours change ([`cl_parse.c`](cl_parse.c.md)), rather than from the original's fixed player colours. Otherwise unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
