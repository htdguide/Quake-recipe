# QW/client/vid_win.c

> The Windows software-video backend.

**Needs** — as [`vid_win.c`](../../WinQuake/vid_win.c.md)
**Used by** — as [`vid_win.c`](../../WinQuake/vid_win.c.md)
**Tier floor** — as [`vid_win.c`](../../WinQuake/vid_win.c.md)

## Purpose

Read [`vid_win.c`](../../WinQuake/vid_win.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`vid_win.c`](../../WinQuake/vid_win.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Nothing of substance. The lock semantics, the deferred palette and mode changes, and the release-everything-on-focus-loss rule are unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
