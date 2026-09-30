# WinQuake/block8.h

> A generated body of x86 assembly: the surface builder's inner loop, unrolled once per mip level, for an eight-bit destination.

**Needs** — [`asm_draw.h`](asm_draw.h.md) · [`quakeasm.h`](quakeasm.h.md) · included by [`r_surf.c`](r_surf.c.md)'s assembly counterpart
**Used by** — the accelerated build of [`r_surf.c`](r_surf.c.md)
**Tier floor** — T0 as written; the contract it implements is tier-free

## Purpose

Not a header in any ordinary sense: it is a block of assembly text, included at four different points
with different surrounding definitions, producing four mip-specific versions of one loop. The contract
is [`r_surf.c`](r_surf.c.md)'s.

## State

None; it is code.

## What the loop computes

**Contract** — given a source texture block, a lightmap block, and a destination, produce one 16-by-16
texel block of a lit surface: for each texel, look up its palette index in the texture, combine it with
the interpolated light level, and write the shaded index through the shading table.

```text
# The contract, stated once; see r_surf.c for the full specification.
FOR EACH of the 16 rows IN the block
  light = the row's starting light value          # 8.8 fixed point
  FOR EACH of the 16 texels IN the row
    dest[texel] = colormap[(light BITAND 0xFF00) BITOR texture[texel]]
    light = light + the per-texel light step
  advance light by the per-row step
```

**Invariants** — the lightmap is sampled on a 16-unit grid and the texture at the mip level's own
resolution, so **the ratio between them differs per mip level** — which is exactly why there are four
versions of this loop rather than one parameterized by a shift. At the finest level one lightmap sample
covers sixteen texels; at the coarsest, one covers one.

The shading table lookup takes the light's **high byte** as the row and the texel as the column
([`vid.h`](vid.h.md#the-lighting-table)), which is why the light is 8.8 fixed point: the fractional
part is discarded by the masking, for free.

**Notes** — a rebuild writes one loop with the ratio as a variable and lets the compiler specialize it,
or computes the whole block with vector operations. The four-way unrolling is the 1996 answer to an
inner loop that cannot afford a shift.

The file's existence — assembly in a `.h` — is a build-system artifact: the same text is needed at four
points with different constants, and the assembler has no other inclusion mechanism.
