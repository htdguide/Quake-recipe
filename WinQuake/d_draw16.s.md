# WinQuake/d_draw16.s

> The sixteen-bit span filler.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`asm_draw.h`](asm_draw.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`d_scan.c`](d_scan.c.md), when the display's pixel width is two
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

As [`d_draw.s`](d_draw.s.md).

## `D_DrawSpans16`

**Contract** — identical to the eight-bit filler ([`d_scan.c`](d_scan.c.md#d_drawspans8)) except that
each sampled palette index is expanded through the sixteen-bit shading table
([`vid.h`](vid.h.md#the-lighting-table)) and written as a packed colour, with the destination advancing
two bytes per pixel.

**Invariants** — the subdivision interval, the clamping and the rounding are the same. The only
difference is the destination width.

**Notes** — as with [`block16.h`](block16.h.md), a rebuild that expands the palette once at frame end
keeps a single eight-bit path and deletes this.
