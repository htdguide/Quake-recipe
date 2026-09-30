# WinQuake/d_local.h

> The software rasterizer's private state: the surface cache entry, the per-span gradient setup, the depth buffer, and the mip selection.

**Needs** — [`r_shared.h`](r_shared.h.md) · [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [`cvar.h`](cvar.h.md)
**Used by** — every `d_*.c` file; [`asm_draw.h`](asm_draw.h.md) mirrors the span record
**Tier floor** — T1 as written

## Purpose

Declares three things the rasterizer turns on: the **surface cache entry**, whose invalidation rules
are the whole of how lighting updates reach the screen; the **per-span gradient variables**, which are
the perspective-correct texture mapping's entire state; and the **depth buffer**, which exists only
for models, sprites and particles — not for the world.

## State

```text
CONSTANT scanbufferpad = 0x1000      # slack for a 2048-wide scan subdivided
                                     # every 8 pixels
CONSTANT r_sky_smask = 0x007F0000    # the sky's 128-texel wrap, in 16.16
CONSTANT r_sky_tmask = 0x007F0000
CONSTANT ds_span_list_end = -128
CONSTANT surfcache_size_at_320x200 = 614400   # 600 KB

RECORD SurfCache                # one cached, lit, mipped surface
  next     : SurfCache          # the cache's address-ordered ring
  owner    : pointer to SurfCache   # back to the MSurface's slot; NOTHING
                                    # means this chunk is free
  lightadj : int[4]             # the light levels it was built with
  dlight   : int                # the dynamic lights it was built with
  size     : int                # including this header
  width, height : int
  mipscale : real
  texture  : Texture            # which texture it was built from
  data     : bytes              # width * height texels

RECORD SSpan                    # a sprite's span; no gradient, no depth
  u, v, count : int
```

**Invariants** — **the cache entry records what it was built from, and that is how it is
invalidated.** An entry whose stored light levels differ from the surface's current ones, or whose
dynamic-light set differs, or whose texture differs from the current animation frame, is rebuilt. So a
flickering light invalidates every surface it touches, every time it changes level — which is why a
room full of flickering lights is expensive, and why the light style levels are 8.8 fixed point rather
than continuous: fewer distinct values means fewer rebuilds.

The `owner` back-pointer is the same idea as the general cache's handle
([`zone.h`](zone.h.md)): the surface cache must be able to tell a surface that its cached form is gone.
A null owner marks free space.

The cache is a **rover-allocated ring**, not a least-recently-used store: allocation walks forward from
where the last one ended and takes whatever it finds, evicting as it goes. That is why thrashing is
possible and why the renderer reports it ([`render.h`](render.h.md)).

600 kilobytes at the base resolution, scaled by the video backend for larger ones
([`render.h`](render.h.md#surface-cache)).

## The span gradient

```text
VARIABLE d_sdivzstepu, d_tdivzstepu, d_zistepu : real    # per pixel across
VARIABLE d_sdivzstepv, d_tdivzstepv, d_zistepv : real    # per scanline down
VARIABLE d_sdivzorigin, d_tdivzorigin, d_ziorigin : real # at the screen origin
VARIABLE sadjust, tadjust : fixed16      # the surface's texture-space offset
VARIABLE bbextents, bbextentt : fixed16  # its clamping bounds
```

**Invariants** — this is perspective-correct texture mapping in nine numbers. The quantities
interpolated linearly across the screen are **s/z, t/z and 1/z**, not s, t and z — because those three
*are* linear in screen space for a planar surface. Recovering the texture coordinate at a pixel is then
one division: `s = (s/z) / (1/z)`.

The division is the expensive part, and the rasterizer performs it **once every sixteen pixels**,
interpolating linearly in between ([`d_scan.c`](d_scan.c.md)). That subdivision interval is the single
most visible approximation in the renderer: it is why textures on surfaces at grazing angles show a
faint sawtooth.

The clamping bounds exist because the interpolation between divisions can overshoot the surface's real
extent, and sampling outside it would read another surface's texels
([`model.c`](model.c.md#mod_loadfaces) sets the liquid surfaces' extents enormous precisely so this
clamp does not bite them).

## The depth buffer

```text
VARIABLE d_pzbuffer   : int (16-bit)[]      # the depth buffer, 16 bits per pixel
VARIABLE d_zrowbytes, d_zwidth : int
VARIABLE zspantable   : list<pointer>[1024] # a row address per scanline
VARIABLE d_pscantable : list<int> ;  d_scantable : int[1024]   # likewise for
                                                               # the colour buffer
```

**Invariants** — **there is a depth buffer, and the world does not use it.** The world is drawn by the
edge sorter with no depth test at all. The depth buffer is written *by* the world pass and read by the
model, sprite and particle passes — which is how a monster can be correctly occluded by a wall without
the wall pass paying for a depth test. That asymmetry is the architecture's cleverest single trick,
and a rebuild using a depth buffer throughout loses nothing but does more work.

Depth is stored as **one over z scaled into 16 bits**, consistent with the reciprocal convention
([`d_iface.h`](d_iface.h.md)).

The two scan tables are precomputed row addresses, so a rasterizer reaches scanline *v* with one
indexed load rather than a multiply. Incidental.

## Mip selection

```text
VARIABLE scale_for_mip : real
VARIABLE d_minmip      : int              # never use a level finer than this
VARIABLE d_scalemip    : real[3]          # the three level boundaries
```

**Contract** — `D_MipLevelForScale` takes a texels-per-pixel scale and returns which of the four mip
levels to use.

**Invariants** — the boundaries are three thresholds against the scale, so the choice is two or three
comparisons. The minimum level is a quality control: raising it forces coarser textures everywhere and
is how the engine is made to fit a smaller surface cache.

## Entry points

**Contract** — the span fillers, one per destination width and per surface kind: the ordinary
eight- and sixteen-bit fillers, the depth-only filler, the liquid warp filler, the sprite filler with
its transparency test, and the two sky fillers. `D_CacheSurface` returns a surface's cached form for a
mip level, building it if absent. `R_ShowSubDiv` is a diagnostic that tints each subdivision interval.

**Invariants** — the span fillers are the [Seam: Vectorized inner
loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)'s largest members, and each has an
assembly twin. The *sprite* filler needs a transparency test per pixel while the surface fillers do
not, which is why it is a separate routine rather than a flag.

`prealspandrawer` and `d_drawspans` are function-pointer indirections that let the filler be swapped at
run time — for the subdivision diagnostic, and for the eight- versus sixteen-bit choice.
