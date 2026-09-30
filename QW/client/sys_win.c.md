# QW/client/sys_win.c

> The Windows system layer and entry point.

**Needs** — as [`sys_win.c`](../../WinQuake/sys_win.c.md)
**Used by** — as [`sys_win.c`](../../WinQuake/sys_win.c.md)
**Tier floor** — as [`sys_win.c`](../../WinQuake/sys_win.c.md)

## Purpose

Read [`sys_win.c`](../../WinQuake/sys_win.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`sys_win.c`](../../WinQuake/sys_win.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The frame loop moved into [`cl_main.c`](cl_main.c.md), so this file is the platform operations plus a thin entry point. The clock's monotonicity guard is unchanged and matters more, because the channel's rate budget and every timeout read it.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
