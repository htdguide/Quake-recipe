# WinQuake/sys_wind.c

> The Windows dedicated-server system layer: the same interface without any of the display, input or floating-point handling, driven from a console.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`sys.h`](sys.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — the dedicated-server build; owns its entry point
**Tier floor** — none

## Purpose

[`sys_win.c`](sys_win.c.md) with everything the server does not need removed: no window, no message pump, no
floating-point control, no code patching, no display restoration on error. Read that twin for the shape.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What is worth recording

**The server's frame loop blocks on console input with a timeout** rather than running as fast as it can, so an idle server
costs nothing. The timeout is what bounds how long a packet waits — the loop wakes, runs a frame, and the frame reads the
network. A rebuild should instead wait on the socket *and* the console together, which is both simpler and more responsive;
the recipe records that the original does not, and that the timeout therefore sets the server's worst-case latency.

**A fatal error prints and exits with no display to restore**, which is the whole of the difference in the error path.

**Notes** — the existence of this file plus [`vid_null.c`](vid_null.c.md), [`snd_null.c`](snd_null.c.md),
[`in_null.c`](in_null.c.md) and [`cd_null.c`](cd_null.c.md) is the engine's proof that the server half stands alone.
