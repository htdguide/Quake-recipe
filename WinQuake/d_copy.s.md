# WinQuake/d_copy.s

> Copies the drawing surface to a VGA display, in planar and linear modes.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`d_local.h`](d_local.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — [`vid_vga.c`](vid_vga.c.md) and [`vid_dos.c`](vid_dos.c.md)
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

## `VGA_UpdatePlanarScreen`, `VGA_UpdateLinearScreen`

**Contract** — copy a rectangle of the off-screen surface to the display. The linear form is a strided
block copy. The planar form writes to a display whose four colour planes are separately addressed, so it
must write each pixel to the plane its column selects.

**Invariants** — the planar form exists for the VGA's ModeX layouts, where consecutive pixels live in
different memory planes selected by an output port. So the copy is four passes, one per plane, each
touching every fourth column. That is not an optimization — it is the only way to write those modes.

**Notes** — the *problem* this solves is [Seam: Framebuffer
surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface)'s flush operation on a display whose pixel
addressing is not linear. Every surviving display is linear, so a rebuild implements the linear form and
deletes the other.
