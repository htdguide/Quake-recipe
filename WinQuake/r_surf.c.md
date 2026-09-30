# WinQuake/r_surf.c

> Builds a lit surface: accumulates the static lightmaps and every dynamic light into a block of light values, then walks the surface in 16-texel blocks bilinearly interpolating that light across the texture.

**Needs** — [`r_local.h`](r_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [`client.h`](client.h.md) · [`vid.h`](vid.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops) ([`surf8.s`](surf8.s.md), [`surf16.s`](surf16.s.md), [`block8.h`](block8.h.md))
**Used by** — [`d_surf.c`](d_surf.c.md) calls it when a cached surface is missing or invalid — the callback up ([`d_iface.h`](d_iface.h.md#the-call-up))
**Tier floor** — none

## Purpose

All of this engine's lighting is here. Two stages: build a small array of light values, one per 16-unit
lightmap sample; then expand that array across the surface's texels, interpolating in both axes, and combine
each texel with its light through the shading table.

The interpolation is what makes lighting look smooth despite one sample per 16 world units, and the
sixteen-block structure is what makes it cheap.

## State

```text
VARIABLE r_drawsurf : DrawSurf               # the request (d_iface.h)
VARIABLE blocklights : int[324]              # 18 by 18 light values, 8.8 fixed
VARIABLE r_lightptr : pointer                # into blocklights
VARIABLE r_lightwidth : int                  # samples per lightmap row
VARIABLE lightleft, lightright, lightdelta : int
VARIABLE lightleftstep, lightrightstep, lightdeltastep : int
VARIABLE blocksize, blockdivshift, blockdivmask : int
VARIABLE r_source, r_sourcemax, pbasesource : bytes
VARIABLE sourcetstep, r_stepback, surfrowbytes : int
VARIABLE r_numhblocks, r_numvblocks : int
VARIABLE prowdestbase : pointer
VARIABLE surfmiptable : handler[4]           # the four mip-specific block
                                            # drawers
```

**Invariants** — the light block is **18 by 18**, which bounds the surface at 256 by 256 texels: a 256-unit
surface has 17 lightmap samples per axis ([`model.c`](model.c.md#calcsurfaceextents) caps extents at 256) and
the eighteenth is slack. That is why the map compiler subdivides larger faces.

Light values are **8.8 fixed point**, so a sample has 256 fractional steps — which is what lets the
interpolation be smooth and what makes the shading-table lookup a mask
([`vid.h`](vid.h.md#the-lighting-table)).

## `R_BuildLightMap`

**Contract** — fills the light block for the requested surface: clears to the ambient floor, adds each active
light style's baked lightmap scaled by that style's current level, adds every dynamic light, then bounds,
**inverts** and shifts every value into the shading table's index range. With full-bright enabled or a map
with no lighting data, clears to zero instead.

```text
FUNCTION r_build_light_map()
  surf = the request's surface
  smax = (surf.extents[0] SHIFTED RIGHT 4) + 1      # samples per axis
  tmax = (surf.extents[1] SHIFTED RIGHT 4) + 1
  size = smax * tmax

  IF full-bright OR the world has no lighting data
    zero the whole block ;  RETURN

  # 1. The ambient floor, from the tunable.
  FOR EACH i IN 0..size-1  blocklights[i] = r_refdef.ambientlight SHIFTED LEFT 8

  # 2. Each light style's baked lightmap, scaled by its CURRENT level.
  IF the surface has lightmap samples
    lightmap = surf.samples
    FOR EACH style slot maps WHILE surf.styles[maps] != 255
      scale = r_drawsurf.lightadj[maps]               # 8.8; the style's level
      FOR EACH i  blocklights[i] = blocklights[i] + lightmap[i] * scale
      lightmap = lightmap + size                     # the next layer

  # 3. Dynamic lights.
  IF a dynamic light touched this surface THIS frame  r_add_dynamic_lights()

  # 4. Bound, INVERT and shift into the shading table's index space.
  FOR EACH i
    t = (255*256 - blocklights[i]) SHIFTED RIGHT (8 - 6)
    IF t < 64  t = 64
    blocklights[i] = t
```

**Invariants** — five things.

**The light value is inverted.** The shading table's rows go from bright to dark, so a large light value must
become a small index. `255*256` minus the accumulated light is that inversion, and getting it backwards makes
lit surfaces black.

**The shift is by 2**, from 8.8 fixed point into 6.10 — because the table has 64 rows
([`vid.h`](vid.h.md)) and the row is selected by the value's high byte
([`block8.h`](block8.h.md)). So after the shift, the high byte holds a row index in 0..63 and the low byte is
the interpolation fraction.

**The floor of 64 is one shading row**, which prevents the brightest row (index 0) from ever being selected —
because that row is reserved and using it over-brightens. A rebuild must keep the floor or lit surfaces
oversaturate.

**Each light style contributes a separate baked layer**, stacked in the lightmap data, scaled by that style's
current animation level ([`r_light.c`](r_light.c.md#r_animatelight)). So a surface lit by a steady lamp and a
flickering torch has two layers and only the second's scale changes — which is why only surfaces touching the
flickering light are invalidated.

**The ambient floor is added before everything**, so raising it lightens the whole world uniformly. That is
the tunable players use to make dark rooms playable.

## `R_AddDynamicLights`

**Contract** — for each dynamic light flagged as touching the surface, projects the light onto the surface's
plane, converts the projection into lightmap sample coordinates, and adds a falloff contribution to every
sample within reach.

```text
FUNCTION r_add_dynamic_lights()
  surf = the request's surface
  FOR EACH light lnum WHOSE bit IS SET IN surf.dlightbits
    rad = the light's radius
    dist = the signed distance FROM the light TO the surface's plane
    rad = rad - |dist|                           # reach, reduced by standoff
    IF rad < the light's minimum  CONTINUE       # too far to matter
    minlight = rad - the light's minimum

    # Project the light onto the plane, then into the surface's texture space.
    impact = the light's origin, moved onto the plane
    local[0] = dot(impact, tex.vecs[0]) + tex.vecs[0][3] - surf.texturemins[0]
    local[1] = dot(impact, tex.vecs[1]) + tex.vecs[1][3] - surf.texturemins[1]

    FOR EACH sample row t, column s
      sd = |local[0] - s*16| ;  td = |local[1] - t*16|
      # An approximation of the Euclidean distance: the larger plus HALF the
      # smaller.
      dist = max(sd,td) + (min(sd,td) SHIFTED RIGHT 1)
      IF dist < minlight
        blocklights[t*smax + s] = blocklights[...] + (minlight - dist) * 256
```

**Invariants** — three decisions.

**The distance is approximated as the larger component plus half the smaller.** That is a cheap octagonal
approximation of a circle, and it is why a dynamic light's falloff in this game is faintly octagonal rather
than round. A rebuild using a real distance looks slightly different and slightly better.

**The falloff is linear in that distance**, not inverse-square, so a light is a cone rather than a physical
point source. Deliberate: it keeps the lit region bounded exactly at the radius.

**Only lights whose bit is set in the surface's dynamic-light set are considered**, and that set was built by
the light-marking pass ([`r_light.c`](r_light.c.md#r_marklights-r_pushdlights)). So the cost is proportional to lights
actually near the surface, and the set is a 32-bit integer — the same limit as the light array
([`client.h`](client.h.md)).

## `R_TextureAnimation`

**Contract** — takes a base texture; returns the frame of its animation cycle appropriate to the current time,
following the alternate cycle when the current entity's frame number is non-zero. A cycle with a gap or a
broken link is a fatal error, as is one that does not terminate within a hundred steps.

```text
FUNCTION r_texture_animation(base) -> Texture
  IF currententity.frame != 0 AND base has an alternate cycle
    base = the alternate cycle's first frame
  IF base has no animation  RETURN base
  relative = truncate(cl.time * 10) MODULO base.anim_total
  WHILE relative is outside [base.anim_min, base.anim_max)
    base = base.anim_next
    IF there is none            FAIL WITH "broken cycle"
    IF more than 100 steps      FAIL WITH "infinite cycle"
  RETURN base
```

**Invariants** — **the entity's frame number selects between the two cycles**, which is how a switch or a
powered door toggles its texture ([`model.c`](model.c.md#mod_loadtextures) builds both). One bit of entity
state, two cycles.

**The phase is the clock times ten, modulo the cycle's total**, and the total is in tenths of a second — so
animated textures run at five frames per second with two tenths per frame. Not configurable.

The hundred-step guard turns a malformed cycle into a diagnosable error rather than a hang.

## `R_DrawSurface`

**Contract** — builds one complete lit surface into the request's destination. Computes the light block, then
walks the surface in blocks of 16 texels at mip level 0 (8, 4 or 2 at coarser levels), calling the
mip-specific block drawer for each column of blocks.

```text
FUNCTION r_draw_surface()
  r_build_light_map()
  surfrowbytes = the request's destination stride
  mt = the request's texture
  r_source = the texture's pixels AT the request's mip level

  blocksize     = 16 SHIFTED RIGHT the mip level          # 16, 8, 4, 2
  blockdivshift = 4 - the mip level
  r_lightwidth  = (surf.extents[0] SHIFTED RIGHT 4) + 1
  r_numhblocks  = the destination width  SHIFTED RIGHT blockdivshift
  r_numvblocks  = the destination height SHIFTED RIGHT blockdivshift

  pblockdrawer  = surfmiptable[the mip level]   IF the display is 8-bit
                  the sixteen-bit drawer         OTHERWISE
  horzblockstep = blocksize, doubled for a 16-bit destination

  # The texture wraps, so the starting offset is taken modulo its size.
  # The added multiple of the size guarantees a positive operand.
  soffset  = ((surf.texturemins[0] SHIFTED RIGHT mip) + (smax << 16)) MOD smax
  basetptr = r_source + (((surf.texturemins[1] SHIFTED RIGHT mip)
                          + (tmax << 16)) MOD tmax) * texwidth
  r_sourcemax = r_source + tmax*smax ;  r_stepback = tmax*texwidth

  FOR EACH horizontal block column u
    r_lightptr = blocklights + u              # this column's light samples
    prowdestbase = the destination column
    pbasesource = basetptr + soffset
    CALL pblockdrawer                          # draws a whole COLUMN of blocks
    advance the destination by horzblockstep
    advance soffset, wrapping at the texture's width
```

**Invariants** — five things.

**One block is 16 texels square at mip level 0** and corresponds to exactly one lightmap cell — because
lightmap samples are 16 world units apart and mip level 0 is one texel per unit. At coarser levels the block
shrinks with the texels, so a block is always one lightmap cell. That correspondence is why there are four
block drawers rather than one.

**The texture wraps and the surface does not start at its origin**, so the starting offset is the surface's
texture minimum modulo the texture's size. The added multiple of the size is there purely to make the
operand positive, because the language's modulo truncates.

**The walk is by column of blocks, not by block**, and the drawer handles a whole column — so the light
pointer advances by the lightmap's width inside the drawer and the caller advances it by one sample per
column.

**The destination stride is a global read by the assembly**
([`quakeasm.h`](quakeasm.h.md#the-shared-globals)), which is why it is assigned here rather than passed.

**The source wraps vertically inside the drawer** by comparing against the texture's end and stepping back by
its height — which is how a surface taller than its texture tiles.

## `R_DrawSurfaceBlock8_mip0` and its three siblings

**Contract** — each draws one column of blocks for its mip level, bilinearly interpolating the light across
each block and combining each texel with it through the shading table.

```text
FUNCTION r_draw_surface_block_8_mip0()          # the 16-texel case
  psource = pbasesource ;  prowdest = prowdestbase
  FOR EACH vertical block v
    # The four corner light values of this block: two now, two a lightmap row
    # further on. Interpolate down the left and right edges.
    lightleft  = r_lightptr[0] ;  lightright = r_lightptr[1]
    r_lightptr = r_lightptr + r_lightwidth
    lightleftstep  = (r_lightptr[0] - lightleft)  SHIFTED RIGHT 4
    lightrightstep = (r_lightptr[1] - lightright) SHIFTED RIGHT 4

    FOR EACH of the 16 rows i
      # Interpolate ACROSS, from the right edge toward the left.
      lightstep = (lightleft - lightright) SHIFTED RIGHT 4
      light = lightright
      FOR EACH of the 16 texels b, FROM 15 DOWN TO 0
        prowdest[b] = colormap[(light BITAND 0xFF00) + psource[b]]
        light = light + lightstep
      psource  = psource + sourcetstep
      lightright = lightright + lightrightstep
      lightleft  = lightleft  + lightleftstep
      prowdest = prowdest + surfrowbytes
    IF psource has passed the texture's end  step back by its height
```

**Invariants** — four things.

**The interpolation is bilinear over the block's four corner samples**, with the vertical steps computed
once per block and the horizontal step once per row. Sixteen shifts per block and one addition per texel.

**The shifts are by 4 at mip level 0** — dividing by the block's 16 texels — and by 3, 2 and 1 at the coarser
levels. That is the *only* difference between the four routines, and it is why they are four routines rather
than one: the shift must be an immediate.

**The inner loop runs right to left** (from index 15 down to 0), which is purely a loop-counter convenience.

**The lookup masks the light's high byte and adds the texel**, exactly as
[`block8.h`](block8.h.md) describes — so the shading table's row comes from the light and the column from the
texture, and the fractional light is discarded for free.

The source's own notes ask whether the locals should be locals rather than globals, and whether a single
delta would serve instead of both edge values as the assembly does. Both are incidental.

## `R_DrawSurfaceBlock16`

**Contract** — one routine rather than four, for a sixteen-bit destination, with the mip ratio as a variable
and the lookup through the sixteen-bit shading table.

**Invariants** — that the sixteen-bit path is not specialized per mip level while the eight-bit path is, is
direct evidence the four-way specialization was a measured optimization rather than a requirement
([`surf16.s`](surf16.s.md) notes the same).

## `R_GenTurbTile`, `R_GenTurbTile16`, `R_GenTile`

**Contract** — generate an unlit tiled surface: a 128-by-128 block filled by replicating a 64-by-64 liquid
texture, or the same for a sixteen-bit destination. `R_GenTile` dispatches by the surface's flags.

**Invariants** — used for surfaces carrying the tiled flag
([`model.h`](model.h.md)), which have no lightmap. The tile size of 128 is fixed
([`d_iface.h`](d_iface.h.md)) and is two copies of the liquid texture in each axis, which lets the warp's
wrapping arithmetic stay within one tile.
