# WinQuake/d_draw.s

> The eight-bit span filler and the depth-only span filler: the software renderer's two hottest loops.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`asm_draw.h`](asm_draw.h.md) · [`d_ifacea.h`](d_ifacea.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`d_scan.c`](d_scan.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Reads the nine span-gradient globals and the surface-sampling globals of [`quakeasm.h`](quakeasm.h.md).

## `D_DrawSpans8`

**Contract** — as [`d_scan.c`](d_scan.c.md#d_drawspans8): fill a list of spans from the cached surface,
dividing to recover exact texture coordinates every sixteen pixels and stepping linearly between.

**Invariants** — the sixteen-pixel subdivision interval, the two divisions per interval, and the
fixed-point stepping are all specified in the portable twin and are reproduced here exactly. What the
assembly adds is that the **divide is issued early and other work is done while it completes** — the
1996 x86's divide had a long latency and could overlap with integer work. That scheduling is invisible
in the result.

## `D_DrawZSpans`

**Contract** — as [`d_scan.c`](d_scan.c.md): write only the depth buffer for a list of spans,
interpolating one over depth linearly. Used for the world pass's contribution to the depth buffer that
the model and sprite passes then test against ([`d_local.h`](d_local.h.md#the-depth-buffer)).

**Invariants** — no division at all, because one over depth is linear in screen space. This is the
cheapest loop in the renderer and it is why the world can seed a depth buffer nearly for free.
