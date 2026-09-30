# WinQuake/quakeasm.h

> The assembly's prologue: the architecture switch, the transparency constant, and the declaration of every C global the hand-written inner loops reach.

**Needs** — [`asm_i386.h`](asm_i386.h.md) is usually included alongside it
**Used by** — every `.s` and `.asm` file in the tree
**Tier floor** — T0 as written

## Purpose

An assembly source cannot see C declarations, so every global it reads or writes must be declared
external by hand. This file is that list, and the list *is* the content: it enumerates the shared
mutable state the renderer's inner loops communicate through.

That enumeration is the useful artifact. It says, precisely, which variables constitute the interface
between the renderer's setup code and its rasterizing loops — about forty of them, all globals, none
passed as arguments. A rebuild that wants to pass them instead has this list as its argument set.

## State

Constants and external declarations only.

```text
CONSTANT id386 = whether the target is x86     # derived from the toolchain
CONSTANT transparent_color = 255               # must match d_iface.h
```

## The shared globals

Grouped by what they serve:

**Span gradient and depth** — the nine perspective-interpolation values, the depth buffer's address
and stride, and the depth gradient ([`d_local.h`](d_local.h.md)).

**Liquid warp** — eight variables carrying the current span's warped texture position, step, source
base, destination and remaining count. The warp's inner loop is entirely stateful in these.

**Surface sampling** — the cached surface's address and width, the texture-space offset and the
clamping bounds.

**Destination** — the view buffer's address and the precomputed per-scanline address table.

**Lighting** — the lightmap pointer, the block count, the row width, and the source and destination
bases for the surface builder's two-dimensional loop.

**Sub-model state** — whether a sub-model is currently being drawn, which changes the depth
comparison.

**Invariants** — every one is a **global written by setup code and read by the loop**, so the
interface is entirely by side effect and the loops take no arguments. That is a 1996 calling-convention
decision, and the cost of it is exactly this file.

The surface builder's four bases — source, destination, light pointer and block count — are the state
of a two-dimensional loop that has been *split* across a C caller and an assembly callee: the caller
advances the outer loop and the callee runs the inner one, communicating through these
([`r_surf.c`](r_surf.c.md) and the surface-block routines). A rebuild should write one nested loop and
delete all four.

**Notes** — the file is included by assembly and by the C files that share these globals, and it
switches its own content on whether the hardware renderer is being built, because the hardware build
has no rasterizer to declare these for.
