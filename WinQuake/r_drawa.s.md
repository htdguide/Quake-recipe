# WinQuake/r_drawa.s

> Clips one world polygon edge against the view frustum and emits it into the edge list.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`asm_draw.h`](asm_draw.h.md) · [`d_ifacea.h`](d_ifacea.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`r_draw.c`](r_draw.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Reads the edge and surface layouts of [`asm_draw.h`](asm_draw.h.md).

## `R_ClipEdge`

**Contract** — as [`r_draw.c`](r_draw.c.md#r_clipedge): recursively clip a world-space edge against a
chain of clip planes, then project the surviving segment and add it to the per-scanline edge buckets.

**Invariants** — the recursion, the clip epsilon and the caching of an already-emitted map edge through
its owner field ([`r_shared.h`](r_shared.h.md)) are all specified in the portable twin. The 32-byte
edge record ([`asm_draw.h`](asm_draw.h.md)) is what lets this routine index the edge pool with a shift.

This is the single largest assembly routine in the renderer, because it is on the path of every visible
polygon edge in every frame.
