# QW/client/gl_rsurf.c

> The hardware world renderer: lightmaps packed into pages by a skyline allocator.

**Needs** — as [`gl_rsurf.c`](../../WinQuake/gl_rsurf.c.md)
**Used by** — as [`gl_rsurf.c`](../../WinQuake/gl_rsurf.c.md)
**Tier floor** — as [`gl_rsurf.c`](../../WinQuake/gl_rsurf.c.md)

## Purpose

Read [`gl_rsurf.c`](../../WinQuake/gl_rsurf.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`gl_rsurf.c`](../../WinQuake/gl_rsurf.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Nothing of substance. The packing, the inverted samples, the dirty-rectangle upload and the texture-order drawing are unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
