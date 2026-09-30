# QW/client/r_part.c

> Particles, and the software renderer's network diagnostic display.

**Needs** — as [`r_part.c`](../../WinQuake/r_part.c.md)
**Used by** — as [`r_part.c`](../../WinQuake/r_part.c.md)
**Tier floor** — as [`r_part.c`](../../WinQuake/r_part.c.md)

## Purpose

Read [`r_part.c`](../../WinQuake/r_part.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`r_part.c`](../../WinQuake/r_part.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The particle pool and its effects are unchanged. Added is the software build's version of the packet graph drawn by [`gl_ngraph.c`](gl_ngraph.c.md) in the hardware build — the same three-way distinction between a packet lost, a packet choked locally, and a packet whose delta reference was unusable. Also added are the team-coloured trails the team game uses, driven by two bits of an entity's effect field.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
