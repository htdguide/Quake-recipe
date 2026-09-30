# QW/client/draw.c

> The software renderer's two-dimensional drawing: characters, images, fills and the tiling background, written directly into the framebuffer.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`draw.h`](draw.h.md) · [`vid.h`](vid.h.md) · [`wad.h`](wad.h.md)
**Used by** — [`screen.c`](screen.c.md) · [`sbar.c`](sbar.c.md) · [`menu.c`](menu.c.md) · [`console.c`](console.c.md)
**Tier floor** — T1: it writes bytes into a framebuffer at a stride the platform chooses

## Purpose

Read [`draw.c`](../../WinQuake/draw.c.md) for the substance: the font sheet, the transparent index, the tiling fill derived from screen
coordinates, the dirty-rectangle interaction and the disc indicator that writes to the front buffer.

## State

As [`draw.c`](../../WinQuake/draw.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**A sub-image draw**, which the scoreboard needs ([`sbar.c`](sbar.c.md)).

**The colour-translated draw operates from the kept pixel copy** of the player model, for the menu's live preview, the same as the
hardware path ([`gl_draw.c`](gl_draw.c.md)).

**A character can be drawn into an arbitrary buffer**, not only the framebuffer, which the screen capture needs
([`screen.c`](screen.c.md)).

**Invariants** — the pixel-exact two-dimensional space with its origin at the top left is unchanged, and remains the reason the whole
interface is shared between the two renderers.
