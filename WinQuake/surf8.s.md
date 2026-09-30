# WinQuake/surf8.s

> Builds a lit, mipped surface into the cache: one routine per mip level, for an eight-bit destination.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`asm_draw.h`](asm_draw.h.md) · [`block8.h`](block8.h.md), which supplies the inner loop · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`r_surf.c`](r_surf.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Reads the surface-builder globals of [`quakeasm.h`](quakeasm.h.md).

## The routines

**Contract** — four mip-specific versions of
[`r_surf.c`](r_surf.c.md), plus the floating-point-control bracket and a patching
routine. Each combines a texture block with an interpolated lightmap through the shading table.

**Invariants** — four versions rather than one because the lightmap-to-texel ratio differs per mip level
([`block8.h`](block8.h.md)); the inner loop's text is included from that file at four points.

The bilinear-ish light interpolation, the 8.8 fixed-point light value and the shading-table lookup are all
specified in [`r_surf.c`](r_surf.c.md). The patching routine writes a computed stride into an immediate
operand, which is the second use of [`sys.h`](sys.h.md#sys_makecodewriteable).
