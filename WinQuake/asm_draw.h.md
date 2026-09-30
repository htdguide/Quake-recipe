# WinQuake/asm_draw.h

> The assembly's copy of the software renderer's own layouts: spans, edges, surfaces, clip planes, vertices and the model rasterizer's span package.

**Needs** — nothing; a list of constants
**Used by** — every rasterizing `.s`/`.asm` file: the span fillers, the edge stepper, the model rasterizer, the particle plotter
**Tier floor** — T0 as written

## Purpose

The third and largest of the three assembly layout headers, and the same kind of file as
[`d_ifacea.h`](d_ifacea.h.md) and [`asm_i386.h`](asm_i386.h.md): C records restated as byte offsets,
each with a warning naming the declaration it must match.

Two entries in it are decisions rather than transcription, and those are the content.

## State

Constants only.

```text
CONSTANT near_clip = 0.01     # must match r_local.h
CONSTANT cycle     = 128      # must match r_local.h
```

## The frozen layouts

```text
espan.u = 0 ;  .v = 4 ;  .count = 8 ;  .pnext = 12 ;  size = 16
sspan.u = 0 ;  .v = 4 ;  .count = 8 ;  size = 12

# The model rasterizer's per-scanline package: everything one span of a
# gouraud-shaded, textured triangle needs.
spanpackage.pdest = 0 ;  .pz = 4 ;  .count = 8 ;  .ptex = 12 ;  .sfrac = 16
           .tfrac = 20 ;  .light = 24 ;  .zi = 28 ;  size = 32

edge.u = 0 ;  .u_step = 4 ;  .prev = 8 ;  .next = 12 ;  .surfs = 16
    .nextremove = 20 ;  .nearzi = 24 ;  .owner = 28 ;  size = 32

surf.<fields>                 # with SURF_T_SHIFT = 6
```

**Invariants** — two numbers here are load-bearing.

**The edge record is exactly 32 bytes** and the model rasterizer's span package is exactly 32 bytes.
Both are powers of two so that indexing an array of them is a shift. The edge record reaches 32
naturally; the span package reaches it because its eight fields are each four bytes.

**The surface record's shift is 6**, meaning 64 bytes — which is the padding
[`r_shared.h`](r_shared.h.md) applies explicitly. The shift is how the assembly converts between a
surface's pool index and its address, and it is also how the sorter's
"greater address means nearer" comparison works on indices. So the 64-byte size is not merely cache
alignment: **the assembly depends on it being a power of two.** A rebuild that narrows the surface
record to its real 50-odd bytes must change the sorter's index arithmetic.

**Notes** — as with the other two layout headers, a rebuild generates this or deletes it. The two
power-of-two sizes are worth carrying forward as a note on the records themselves, which is where this
twin records them.
