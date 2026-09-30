# QW/client/sys_wina.asm

> A handful of platform helpers in hand-written processor code: the floating-point control word manipulation the renderer needs.

**Needs** — [`quakeasm.h`](quakeasm.h.md)
**Used by** — the Windows build, alongside [`sys_win.c`](sys_win.c.md)
**Tier floor** — T1: it sets processor state the renderer's conversions depend on
[Seam: Vectorized inner loops](../../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)

## Purpose

The assembly counterpart of the floating-point mode handling described in
[`sys_win.c`](../../WinQuake/sys_win.c.md#sys_setfpcw-sys_pushfpcw_sethigh-sys_popfpcw-maskexceptions) — read that twin for why the renderer needs a specific rounding and
precision mode at all.

## State

As [`sys_wina.s`](../../WinQuake/sys_wina.s.md) — the same routines, differing only in syntax.

## The routines

**Contract** — read the current control word, install a modified one, and restore the saved one; and mask the arithmetic exception
traps.

**Invariants** — these exist because the operations have no portable expression, not because they are fast. **A rebuild in a
language with defined real-to-integer conversion semantics needs none of them**, which is the whole point worth recording: this
file is a symptom of the source language's conversion rules, not of the algorithm.

**Notes** — the original keeps the same routines in a different syntax
([`sys_wina.s`](../../WinQuake/sys_wina.s.md)); the same two-copy hazard applies as for the other assembly files here.
