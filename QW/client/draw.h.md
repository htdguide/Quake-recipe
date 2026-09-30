# QW/client/draw.h

> The two-dimensional drawing interface, shared by both renderers.

**Needs** — nothing
**Used by** — [`screen.c`](screen.c.md) · [`sbar.c`](sbar.c.md) · [`menu.c`](menu.c.md) · [`console.c`](console.c.md)
**Tier floor** — none

## Purpose

Read [`draw.h`](../../WinQuake/draw.h.md). The additions are the sub-image draw and the draw-into-a-buffer form
([`draw.c`](draw.c.md)).

**Invariants** — **this interface is what both renderers implement identically** ([`draw.c`](draw.c.md),
[`gl_draw.c`](gl_draw.c.md)), and it is why the interface code is renderer-independent. Keeping it small is what makes that true.
## State

As [`draw.h`](../../WinQuake/draw.h.md); the records are unchanged except where **What differs** says otherwise.

