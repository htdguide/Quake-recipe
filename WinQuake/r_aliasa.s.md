# WinQuake/r_aliasa.s

> Transforms, projects and lights one model's vertices into the rasterizer's fixed-point form.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`d_ifacea.h`](d_ifacea.h.md) · [`asm_draw.h`](asm_draw.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`r_alias.c`](r_alias.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Reads the final-vertex and mesh-descriptor layouts of [`d_ifacea.h`](d_ifacea.h.md).

## `R_AliasTransformAndProjectFinalVerts`

**Contract** — as [`r_alias.c`](r_alias.c.md#r_aliastransformandprojectfinalverts): for each of a
model's vertices, dequantize it, transform it into view space, project it to the screen, look up its
brightness from its normal index, set its clip flags, and store the result as six fixed-point integers.

**Invariants** — the output is entirely fixed point ([`d_iface.h`](d_iface.h.md)) including the light
value, so the rasterizer does no floating-point work at all. The record is padded to 32 bytes so that
indexing is a shift ([`d_ifacea.h`](d_ifacea.h.md)).

The brightness comes from the precomputed table ([`anorm_dots.h`](anorm_dots.h.md)), so lighting a
vertex is one indexed load — which is what makes this loop as tight as it is.
