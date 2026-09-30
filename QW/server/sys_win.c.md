# QW/server/sys_win.c

> The dedicated server's Windows system layer: files, the clock, a console window, and a frame loop that waits on the socket and the console.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`sys.h`](sys.h.md) · [Seam: Operating system services](../../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — the server program; owns the entry point
**Tier floor** — none

## Purpose

The platform layer of [`sys_unix.c`](sys_unix.c.md) over a different operating system — read that twin for the frame loop and the
readiness wait, which are the content. A quarter the size of the original's Windows layer
([`sys_win.c`](../../WinQuake/sys_win.c.md)), and the difference is everything the client needed.

## State

As [`sys_win.c`](../../WinQuake/sys_win.c.md); the records are unchanged except where **What differs** says otherwise.

## What is worth recording

**The console is a real console window**, created by the program, with the server's output going to it and lines read back from it.
That is the server's only local interface.

**No floating-point control, no code patching, no display, no input pump.** Absent, not empty.

**The clock is the same high-resolution counter with the same monotonicity guard** as the original
([`sys_win.c`](../../WinQuake/sys_win.c.md)) — the guard matters as much here, because the server's timeouts and rate budgets all
read it.

**Notes** — worth comparing with [`sys_wind.c`](../../WinQuake/sys_wind.c.md), the original's dedicated-server layer: same job,
and this one waits properly on both inputs.
