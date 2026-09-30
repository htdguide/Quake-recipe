# QW/server/sys.h

> The dedicated server's platform interface, reduced to what a server needs.

**Needs** — nothing
**Used by** — [`sys_unix.c`](sys_unix.c.md) · [`sys_win.c`](sys_win.c.md) · every file that opens a file or reads the clock
**Tier floor** — none

## Purpose

Read [`sys.h`](../../WinQuake/sys.h.md) for the interface.

## State

As [`sys.h`](../../WinQuake/sys.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**The renderer-related operations are gone**: making code writable, the floating-point precision pair, and the input pump. What
remains is files, the clock, printing, the fatal error, the clean exit and a yield.

**Invariants** — that is the measurement worth keeping: **a headless server needs seven platform operations**
([Seam: Operating system services](../../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)). Everything else in the original's
interface exists for the client.
