# WinQuake/d_spr8.s

> The sprite span filler: like the surface filler but with a per-pixel transparency test.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`asm_draw.h`](asm_draw.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — the accelerated build of [`d_sprite.c`](d_sprite.c.md)
**Tier floor** — T0 as written; the contracts it implements are tier-free

## Purpose

Hand-written x86 assembly, selected instead of the portable implementation when the accelerated build is
enabled. Every routine here satisfies a contract stated elsewhere, and this twin records **which**
contract and **what differs**.

A rebuild implements the contracts and ignores this file. The reason to read the page at all is the
"what differs" section: where a hand-written routine is not merely faster but *behaves* differently, that
difference is observable and a rebuild must decide about it.

## State

As [`d_draw.s`](d_draw.s.md), plus the sprite span layout of [`d_local.h`](d_local.h.md).

## `D_SpriteDrawSpans`

**Contract** — as [`d_sprite.c`](d_sprite.c.md#d_spritedrawspans): fill a list of sprite spans with
perspective-corrected sampling, **skipping any texel whose palette index is 255** and depth-testing each
written pixel.

**Invariants** — the transparency test is the whole reason this is a separate routine from the surface
filler, and it costs a compare and branch per pixel — which is why sprites are more expensive per pixel
than walls. The test is against the transparency constant shared with
[`d_iface.h`](d_iface.h.md).

Sprites depth-test but **do not write depth**, so two overlapping sprites blend in draw order rather than
occluding.
