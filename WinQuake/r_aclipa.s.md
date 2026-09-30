# WinQuake/r_aclipa.s

> Clips a model triangle's edge against one of the four screen boundaries.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`d_ifacea.h`](d_ifacea.h.md) · [`asm_draw.h`](asm_draw.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`r_aclip.c`](r_aclip.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

Stateless.

## The four routines

**Contract** — as [`r_aclip.c`](r_aclip.c.md): given two final vertices straddling one screen boundary,
produce the vertex on the boundary by interpolating position, texture coordinate, light and depth
reciprocal.

**Invariants** — one routine per boundary rather than one parameterized by a plane, because clamping to a
screen edge is an interpolation in one coordinate with the other **assigned exactly** — so the four cases
have genuinely different code, not merely different constants. That exactness is what stops a clipped
vertex from landing a fraction outside the view and overrunning the surface.

The four are declared in [`r_local.h`](r_local.h.md) through the clip-plane record's edge fields, which
is how the caller selects among them.
