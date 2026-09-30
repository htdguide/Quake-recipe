# WinQuake/r_edgea.s

> The edge sorter: inserts, removes and steps active edges, and turns edge crossings into spans.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`asm_draw.h`](asm_draw.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`r_edge.c`](r_edge.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Reads the edge, surface and span layouts of [`asm_draw.h`](asm_draw.h.md).

## The routines

**Contract** — each satisfies the same-named contract in [`r_edge.c`](r_edge.c.md): insert a scanline's
new edges into the sorted active list; remove those that ended; advance every active edge's fixed-point
x by its step and re-sort by insertion; and walk the active list emitting a span whenever the topmost
surface changes. Two of the seven are the floating-point-control bracket
([`sys.h`](sys.h.md#floating-point-control)) and one patches a constant.

**Invariants** — this is the heart of the architecture described in
[`r_shared.h`](r_shared.h.md), and the assembly reproduces it exactly, including the depth-gradient
comparison used where two surfaces interpenetrate and the address comparison used where they do not. The
64-byte surface record and its shift of 6 ([`asm_draw.h`](asm_draw.h.md)) are what make the index
arithmetic in the sorter a shift.

The re-sort is an **insertion sort over a nearly-sorted list**, which is linear in practice because
edges rarely cross between adjacent scanlines. That is the property the whole design leans on.
