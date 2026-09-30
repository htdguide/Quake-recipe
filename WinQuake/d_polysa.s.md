# WinQuake/d_polysa.s

> The animated-model rasterizer: gradient setup, recursive triangle subdivision, the shaded span filler, and the point-plotting fallback.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`d_ifacea.h`](d_ifacea.h.md) · [`asm_draw.h`](asm_draw.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`d_polyse.c`](d_polyse.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Reads the mesh descriptor and final-vertex layouts of [`d_ifacea.h`](d_ifacea.h.md).

## The routines

**Contract** — each satisfies the same-named contract in [`d_polyse.c`](d_polyse.c.md): compute a
triangle's texture and light gradients; subdivide it recursively until its spans are small enough to
affine-map without visible error; walk its left edge producing span packages; fill a span with textured,
gouraud-shaded, depth-tested pixels; draw a run of vertices as individual points for distant models; and
draw a triangle without subdivision.

**Invariants** — the **recursive subdivision** is the load-bearing idea and it is specified in the
portable twin: rather than perspective-correcting per span as the world rasterizer does, a model triangle
is split until affine mapping is close enough. That is cheaper for the small triangles a model is made of
and it is why models and the world use different rasterizers at all.

## What differs

**Invariants** — two of the routines here are a **self-modifying pair**: one writes a computed shift
count into an immediate operand of the span filler and the other restores it, which is why
[`sys.h`](sys.h.md#sys_makecodewriteable) exists. The *problem* being solved is a shift count that is
constant for a whole model but not at compile time. A rebuild passes it in a register.

The patching routine's counterpart in the portable build is a no-op, so the two builds differ in nothing
observable.
