# WinQuake/sys_sun.c

> The workstation system layer: the Unix layer with a handle table and the platform's own clock.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`sys.h`](sys.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — everything, through [`sys.h`](sys.h.md); owns the entry point
**Tier floor** — none

## Purpose

A near-duplicate of [`sys_linux.c`](sys_linux.c.md) — read that twin. It differs in using a fixed handle table like
[`sys_win.c`](sys_win.c.md)'s rather than passing the platform's descriptors through, in reporting a file's modification
time (which the archive layer uses to decide whether a cached file is stale), and in the clock call it uses.

**Notes** — recorded because the recipe mirrors the tree. A rebuild has one Unix system layer.
## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

