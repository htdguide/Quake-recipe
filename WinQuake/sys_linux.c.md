# WinQuake/sys_linux.c

> The Unix system layer and entry point: file operations over the platform's own, a clock from the interval timer, and a frame loop that yields when there is nothing to do.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`sys.h`](sys.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — everything, through [`sys.h`](sys.h.md); owns the entry point
**Tier floor** — none

## Purpose

The shortest and cleanest of the system layers, and the best one to read first: with a real operating system underneath,
the file interface is a thin wrapper, there is no memory locking, no stack guard and no code patching. Read
[`sys_win.c`](sys_win.c.md) for the shape of the interface and the reasoning behind the clock and the outer loop.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What is worth recording

**The heap is taken as one allocation at startup**, sized from the command line with a default, exactly as everywhere else.
That is the invariant that survives every platform.

**Code patching is not available and not needed**, because this build uses the portable inner loops. The operation that
makes code writable is a no-op, which is the direct evidence that the assembly is a **pluggable** seam
([SYSTEM-REQUIREMENTS](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)) and not a requirement.

**The floating-point precision operations are no-ops** for the same reason.

**A fatal error prints to the terminal after shutting the display down**, and the shutdown is guarded against re-entry.

**The console can be read while the game runs**, non-blocking, which is what makes a dedicated server usable from a shell
and is why the input operation reports "no line yet" rather than blocking.

**A debug log and a fault handler** write to a file, which is the platform's stand-in for a debugger on a user's machine.

**Notes** — a rebuild on any modern platform will look like this file, not like the other two. The useful comparison is the
list of things this one does *not* have to do.
