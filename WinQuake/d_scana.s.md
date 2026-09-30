# WinQuake/d_scana.s

> The liquid-surface span filler: samples a texture through a scrolling sine warp.

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

Reads the eight liquid-warp globals of [`quakeasm.h`](quakeasm.h.md).

## `D_DrawTurbulent8Span`

**Contract** — as [`d_scan.c`](d_scan.c.md#turbulent8): fill a span by sampling a 64-by-64 texture at
coordinates displaced by a sine of the other coordinate and the time, wrapping within the texture.

**Invariants** — the texture is 64 by 64 and the cycle is 128 ([`d_iface.h`](d_iface.h.md)), so both
the sine index and the texture wrap are **masks rather than modulos**. The amplitude and speed are in
[`r_local.h`](r_local.h.md). No perspective correction at all, because a liquid surface is drawn with
the tiled flag and has no cached surface ([`model.h`](model.h.md)).
