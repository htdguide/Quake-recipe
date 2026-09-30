# WinQuake/surf16.s

> The surface builder for a sixteen-bit destination.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`asm_draw.h`](asm_draw.h.md) · [`block16.h`](block16.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`r_surf.c`](r_surf.c.md), when the display's pixel width is two
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

As [`surf8.s`](surf8.s.md).

## The routines

**Contract** — as [`r_surf.c`](r_surf.c.md#r_drawsurfaceblock16), with the shaded index expanded through
the sixteen-bit shading table and the destination advancing two bytes per texel.

**Invariants** — **one routine, not four.** The sixteen-bit path does not specialize per mip level,
because it was added later and its performance mattered less. So the mip ratio is a variable here and an
unrolled constant in the eight-bit version — direct evidence that the four-way specialization was a
measured optimization rather than a necessity.
