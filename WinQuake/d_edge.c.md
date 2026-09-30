# WinQuake/d_edge.c

> Consumes the span list: chooses each surface's mip level, computes its perspective gradient, fetches its cached lit texture, and dispatches to the right span filler.

**Needs** — [`d_local.h`](d_local.h.md) · [`r_local.h`](r_local.h.md) · [`r_shared.h`](r_shared.h.md) · [`model.h`](model.h.md) · [`client.h`](client.h.md) · [`quakedef.h`](quakedef.h.md)
**Used by** — [`r_edge.c`](r_edge.c.md) calls it once per span-list flush
**Tier floor** — none

## Purpose

The bridge between the sorter and the pixel loops. For each surface that produced spans, it answers four
questions — which mip level, what is the texture-space gradient, where is the lit texture, and which
filler — and then calls. The four kinds of surface (sky, background, liquid, ordinary) take four different
paths.

The gradient computation is the substantial content: it is where a surface's plane, texture axes and mip
level become the nine numbers the fillers step.

## State

```text
VARIABLE miplevel : int                       # the surface being drawn
VARIABLE scale_for_mip : real                 # the larger of the two projection
                                              # scales
VARIABLE screenwidth : int                    # the destination's stride
VARIABLE transformed_modelorg : vec3          # the view position in the current
                                              # model's space, in view axes
VARIABLE ubasestep, errorterm, erroradjustup, erroradjustdown, vstartscan : int
```

## `D_MipLevelForScale`

**Contract** — takes a texels-per-pixel scale; returns 0 through 3 by three comparisons against the
threshold table, then clamps to the configured minimum.

```text
FUNCTION d_mip_level_for_scale(scale) -> int
  level = 0 IF scale >= d_scalemip[0]
          1 IF scale >= d_scalemip[1]
          2 IF scale >= d_scalemip[2]
          3 OTHERWISE
  RETURN max(level, d_minmip)
```

**Invariants** — the thresholds come from [`d_init.c`](d_init.c.md)'s base table scaled by a tunable, so a
player can bias the whole level selection coarser to fit a smaller surface cache. The minimum clamp is the
hard version of the same control.

## `D_CalcGradients`

**Contract** — takes a surface; computes the nine gradient values, the texture-space offset and the
clamping bounds that the span fillers step. Must be called after the mip level is chosen and the view is
transformed into the surface's model space.

```text
FUNCTION d_calc_gradients(pface)
  mipscale = 1 / (1 SHIFTED LEFT miplevel)      # 1, 1/2, 1/4, 1/8

  # The texture's two axes, brought into VIEW space.
  p_saxis = transform_vector(pface.texinfo.vecs[0])
  p_taxis = transform_vector(pface.texinfo.vecs[1])

  # s/z and t/z change per screen pixel by the axis's view-space x, scaled by
  # the inverse projection scale and the mip ratio.
  t = xscaleinv * mipscale
  d_sdivzstepu = p_saxis[0] * t ;  d_tdivzstepu = p_taxis[0] * t
  t = yscaleinv * mipscale
  d_sdivzstepv = -p_saxis[1] * t ;  d_tdivzstepv = -p_taxis[1] * t

  # The value at the screen ORIGIN, derived by stepping back from the centre.
  d_sdivzorigin = p_saxis[2]*mipscale - xcenter*d_sdivzstepu
                                      - ycenter*d_sdivzstepv
  d_tdivzorigin = p_taxis[2]*mipscale - xcenter*d_tdivzstepu
                                      - ycenter*d_tdivzstepv

  # The offset from the surface's own texture space to the cached block's.
  p = transformed_modelorg SCALED BY mipscale
  t = 65536 * mipscale
  sadjust = round(dot(p, p_saxis) * 65536)
            - ((pface.texturemins[0] SHIFTED LEFT 16) SHIFTED RIGHT miplevel)
            + pface.texinfo.vecs[0][3] * t
  tadjust = ...the same for t

  # The clamping bounds, one short of the real extent.
  bbextents = ((pface.extents[0] SHIFTED LEFT 16) SHIFTED RIGHT miplevel) - 1
  bbextentt = ((pface.extents[1] SHIFTED LEFT 16) SHIFTED RIGHT miplevel) - 1
```

**Invariants** — five load-bearing details.

**The vertical steps are negated** because screen y increases downward while the view basis's y increases
upward. Getting the sign wrong flips every texture vertically.

**The gradient is expressed at the screen origin, not at the view centre**, so the fillers can evaluate it
from a raw pixel coordinate with two multiplies and no offset. The subtraction of the centre times the
steps is that rebasing.

**The mip level scales both the steps and the offset**, because a coarser mip level's texels are larger:
the same screen distance covers fewer of them. The shift-right of the minimum and extent by the mip level
is the same scaling in fixed point.

**The offset includes the view position dotted with the texture axes**, which is what anchors the texture
to the world rather than to the screen. For a sub-model the view position must first be brought into that
model's space, which is why the caller transforms it before calling.

**The clamping bound is one short of the extent** — the comment calls it minus an epsilon — so that the
fillers' interpolation can never index the last texel plus one. Combined with the fillers' low clamp
([`d_scan.c`](d_scan.c.md#d_drawspans8)), the texture sample is provably inside the block.

## `D_DrawSolidSurface`

**Contract** — fills a surface's spans with one palette index, writing four bytes at a time where
alignment allows.

**Invariants** — used for the background and for the flat-shading diagnostic.

## `D_DrawSurfaces`

**Contract** — walks every surface that produced spans and draws it. Establishes the view position in
world-model space once, then per surface: copies the surface's depth gradient into the globals, and takes
one of five paths.

```text
FUNCTION d_draw_surfaces()
  currententity = the world
  transformed_modelorg = transform_vector(modelorg)
  world_transformed_modelorg = transformed_modelorg      # saved for restoring

  FOR EACH surface s FROM &surfaces[1] TO surface_p-1
    IF s has no spans  CONTINUE
    d_zistepu = s.d_zistepu ;  d_zistepv = s.d_zistepv
    d_ziorigin = s.d_ziorigin

    IF the flat-shading diagnostic is on
      d_draw_solid_surface(s, the low byte of s.data) ;  d_draw_z_spans(s.spans)
      CONTINUE

    IF s has the sky flag
      IF the sky texture is stale  regenerate it
      d_draw_sky_scans_8(s.spans) ;  d_draw_z_spans(s.spans)

    ELSE IF s has the background flag
      # Place the background effectively at infinity: a CONSTANT depth
      # reciprocal, slightly negative so nothing can be behind it.
      d_zistepu = 0 ;  d_zistepv = 0 ;  d_ziorigin = -0.9
      d_draw_solid_surface(s, the clear colour) ;  d_draw_z_spans(s.spans)

    ELSE IF s has the turbulent flag
      pface = s.data ;  miplevel = 0                     # liquids never mip
      cacheblock = the texture's mip 0 ;  cachewidth = 64
      IF s is in a sub-model  enter that model's space    # see below
      d_calc_gradients(pface)
      turbulent_8(s.spans) ;  d_draw_z_spans(s.spans)
      IF s is in a sub-model  restore the world's space

    ELSE                                                  # an ordinary surface
      IF s is in a sub-model  enter that model's space
      pface = s.data
      miplevel = d_mip_level_for_scale(s.nearzi * scale_for_mip
                                       * pface.texinfo.mipadjust)
      cache = d_cache_surface(pface, miplevel)            # builds it if absent
      cacheblock = cache.data ;  cachewidth = cache.width
      d_calc_gradients(pface)
      CALL the selected span filler ON s.spans
      d_draw_z_spans(s.spans)
      IF s is in a sub-model  restore the world's space
```

Entering and restoring a sub-model's space:

```text
# Enter:
currententity = s.entity
local_modelorg = r_origin - currententity.origin
transformed_modelorg = transform_vector(local_modelorg)
r_rotate_bmodel()                     # rotates the view basis and re-derives
                                      # the frustum

# Restore:
currententity = the world
transformed_modelorg = world_transformed_modelorg
vpn, vup, vright = their base copies
modelorg = its base copy
r_transform_frustum()
```

**Invariants** — six things here matter.

**Every surface writes the depth buffer**, including the sky and the background. That is what makes the
model and sprite passes' depth tests correct everywhere
([`d_local.h`](d_local.h.md#the-depth-buffer)).

**The background's depth reciprocal is a negative constant.** One over depth is normally positive and
larger when nearer, so −0.9 is behind everything expressible. That is how the background occludes nothing
and is occluded by everything, with no special case in the depth test.

**Liquids never mip**, always using level 0 with a fixed width of 64. Their warp already destroys the
detail a mip would remove, and mipping would make the warp's wrapping arithmetic wrong.

**The mip level is chosen from the surface's nearest depth reciprocal**, times the projection scale, times
the surface's own texture-scale compensation ([`model.c`](model.c.md#mod_loadtexinfo)). So one surface gets
one mip level for the whole frame even if it spans a large depth range — which is why a long floor
visibly changes mip level as you walk, in one step, rather than continuously.

**The surface cache is consulted per surface per frame**, and builds the lit texture on demand
([`d_surf.c`](d_surf.c.md)). That call is where the whole lighting pipeline hangs off the rasterizer.

**Entering a sub-model's space rotates the view basis and re-derives the frustum, per surface**, and
restores it afterwards. The source marks this four times as something that should be hoisted out of the
loop. It is genuinely expensive — a sub-model with fifty visible faces pays for fifty frustum
derivations — and a rebuild should group surfaces by entity and transform once. The *behaviour* is
unaffected.
