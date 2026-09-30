# QW/client/gl_draw.c

> The texture manager and the hardware two-dimensional layer.

**Needs** — as [`gl_draw.c`](../../WinQuake/gl_draw.c.md)
**Used by** — as [`gl_draw.c`](../../WinQuake/gl_draw.c.md)
**Tier floor** — as [`gl_draw.c`](../../WinQuake/gl_draw.c.md)

## Purpose

Read [`gl_draw.c`](../../WinQuake/gl_draw.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`gl_draw.c`](../../WinQuake/gl_draw.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Added are the sub-image draw the scoreboard needs ([`sbar.c`](sbar.c.md)) and a per-player skin upload path that takes a downloaded image rather than only a model's own skin ([`skin.c`](skin.c.md)). The scrap sheets, the palette expansion, the mip generation and the pixel-exact orthographic mode are unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
