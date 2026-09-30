# WinQuake/block16.h

> The same generated assembly loop as its eight-bit sibling, for a sixteen-bit destination.

**Needs** — [`asm_draw.h`](asm_draw.h.md) · [`quakeasm.h`](quakeasm.h.md)
**Used by** — the accelerated build of [`r_surf.c`](r_surf.c.md), when the display's pixel width is two
**Tier floor** — T0 as written

## Purpose

Identical in structure to [`block8.h`](block8.h.md) and identical in contract to
[`r_surf.c`](r_surf.c.md#r_drawsurfaceblock16). The only difference is the final step.

## State

Data; no run-time state.

## What differs from the eight-bit version

**Contract** — where the eight-bit loop writes the shaded palette index directly, this one uses it to
index the **sixteen-bit** shading table ([`vid.h`](vid.h.md)) and writes a packed colour value.

```text
# The last line of the inner loop becomes:
dest[texel] = colormap16[(light BITAND 0xFF00) BITOR texture[texel]]
              # a 16-bit packed colour rather than a palette index
```

**Invariants** — the destination advances by two bytes per texel, so every address computation in the
loop is scaled. That is the whole of the difference and it is why the file is a near-copy rather than a
parameterization.

**Notes** — the sixteen-bit path exists for backends whose surface is 16-bit
([`vid_sunxil.c`](vid_sunxil.c.md) and one Windows mode). Every asset is converted to the destination
width at load ([`model.c`](model.c.md#mod_loadaliasskin-mod_loadaliasskingroup)), which is why the two paths do not coexist in
one session.

A rebuild targeting a modern surface should expand the palette to 32-bit once, at the end of the frame,
and keep a single 8-bit rasterizing path — which deletes this file, its sibling, and the pixel-width
switch throughout the loaders.
