# QW/client/gl_screen.c

> The hardware frame composer.

**Needs** — as [`gl_screen.c`](../../WinQuake/gl_screen.c.md)
**Used by** — as [`gl_screen.c`](../../WinQuake/gl_screen.c.md)
**Tier floor** — as [`gl_screen.c`](../../WinQuake/gl_screen.c.md)

## Purpose

Read [`gl_screen.c`](../../WinQuake/gl_screen.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`gl_screen.c`](../../WinQuake/gl_screen.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The loading plaque is gone, for the same reason as in [`screen.c`](screen.c.md): a level change does not stop the world. The frame-rate display and the reduced screen capture are added. The two platform frame calls remain the whole per-frame platform surface.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
