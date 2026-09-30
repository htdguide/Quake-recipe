# WinQuake/vid_sunx.c

> The same X11 software backend as the general one, adapted for a workstation vendor's displays and window manager conventions.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`vid.h`](vid.h.md) · [`d_local.h`](d_local.h.md) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`host.c`](host.c.md) · [`screen.c`](screen.c.md)
**Tier floor** — none

## Purpose

A near-copy of [`vid_x.c`](vid_x.c.md) — read that twin for the palette-to-pixel table, the per-depth conversion loops,
the shared-memory image and the pointer grab, all of which are the same.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs

A fullscreen path that sets the window's override flag and resizes it to the display, a window title and a video menu
page, a live gamma command, and a default mode chosen from the display's reported depth rather than assumed.

**Notes** — the duplication is real and has no design content. Recorded because the recipe mirrors the tree. A rebuild has
one X backend.
