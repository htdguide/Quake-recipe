# WinQuake/d_parta.s

> The particle plotter: projects one particle and writes a depth-tested square of pixels.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`d_ifacea.h`](d_ifacea.h.md) · [`asm_draw.h`](asm_draw.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`d_part.c`](d_part.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Reads the particle layout of [`d_ifacea.h`](d_ifacea.h.md).

## `D_DrawParticle`

**Contract** — as [`d_part.c`](d_part.c.md#d_drawparticle): transform a particle into view space, reject
it if nearer than the clip distance, project it, choose a square size from its depth, and write that many
pixels of its colour wherever the depth buffer admits them — updating the depth buffer as it goes.

**Invariants** — a particle is a **screen-aligned square whose side is chosen from depth**, in a small
number of discrete sizes, not a scaled sprite. The depth test is per pixel and the write updates the
buffer, so particles occlude each other correctly.

**Notes** — the size quantization is why particles in this game visibly pop between sizes as they recede.
