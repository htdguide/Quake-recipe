# WinQuake/r_local.h

> The software renderer's private interface: the clip-plane record, the model-lighting record, the tuning constants that shape what gets drawn, and every internal entry point.

**Needs** — [`r_shared.h`](r_shared.h.md) · [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [`render.h`](render.h.md) · [`client.h`](client.h.md) (the colour-shift type) · [`cvar.h`](cvar.h.md)
**Used by** — every `r_*.c` and `d_*.c` file; [`host.c`](host.c.md) and [`model.c`](model.c.md) include it for a few constants; compiled out entirely in the hardware build
**Tier floor** — T1 as written

## Purpose

The software renderer's table of contents. Most of it is declarations whose contracts belong to the
implementing files, so this twin documents what is *decided* here: the tuning constants, the two
records, and the reporting variables that make the renderer's limits visible.

The constants are the interesting part. Each one is a threshold that changes what appears on screen,
and several are marked in the source as needing tuning.

## State

### Records

```text
RECORD ALight                   # how to light one animated model
  ambientlight : int            # a floor, applied everywhere
  shadelight   : int            # the directional component's strength
  plightvec    : vec3           # the direction light comes from

RECORD BEdge                    # an edge of a clipped sub-model face
  v     : MVertex[2]
  pnext : BEdge

RECORD AuxVert                  # a model vertex's view-space position
  fv : real[3]

RECORD ClipPlane
  normal    : vec3
  dist      : real
  next      : ClipPlane
  leftedge, rightedge : byte     # which screen edge this plane corresponds to
  reserved  : byte[2]

RECORD BToFPoly                 # a sub-model face deferred to the back-to-front
  clipflags : int                # pass
  psurf     : MSurface
```

**Invariants** — a model is lit by **exactly two numbers and a direction**: an ambient floor and a
directional strength. There is no per-vertex position-dependent lighting, no shadow, and no
attenuation. The direction is sampled once per model from the world's baked lighting
([`r_light.c`](r_light.c.md#recursivelightpoint-r_lightpoint)), and the per-vertex variation comes entirely from the
normal-index table ([`anorm_dots.h`](anorm_dots.h.md)). That is why models in this game are lit
flatly and why they do not cast shadows.

The clip plane's two edge bytes let the clipper know which screen boundary a plane represents, so a
vertex clipped against it can be *clamped* to that boundary rather than interpolated — which is the
optimization that makes screen-edge clipping cheap.

## Tuning constants

```text
CONSTANT alias_base_size_ratio = 1/11
    # Normalizes a model's stored average triangle size so that the player model
    # works out to about one pixel per triangle at the transition distance.
CONSTANT bmodel_fully_clipped  = 0x10
CONSTANT xcentering = 1/2 ;  ycentering = 1/2
CONSTANT clip_epsilon     = 0.001
CONSTANT backface_epsilon = 0.01
CONSTANT dist_not_set     = 98765
CONSTANT near_clip        = 0.01
CONSTANT maxbvertindexes  = 1000     # new vertices when clipping a sub-model
CONSTANT max_btofpolys    = 5000     # deferred sub-model faces
CONSTANT maxaliasverts    = 2000     # vertices in one animated model
CONSTANT alias_z_clip_plane = 5
CONSTANT amp = 8 * 0x10000 ;  amp2 = 3 ;  speed = 20    # the liquid warp
```

**Invariants** — the ones that change what you see.

**The model size ratio** is the threshold at which an animated model stops being rasterized as
triangles and starts being drawn as individual points
([`d_polyse.c`](d_polyse.c.md)). Its derivation — "about one pixel per triangle" — is the honest
statement of when the two look the same. A rebuild that always rasterizes triangles loses the
optimization and gains nothing visible; a rebuild that gets the ratio wrong shows visibly
disintegrating distant monsters.

**The backface epsilon of 0.01** rather than zero is what stops a surface exactly edge-on from
flickering between drawn and culled as the view moves. The clip epsilon of 0.001 plays the same role
at polygon boundaries.

**The near clip at 0.01 world units** is remarkably close, which is why you can push right up against
a wall in this game without seeing through it. It is also why the depth reciprocal can be very large
and why the fixed-point formats have to accommodate that.

**The vertex limit of 2000 per animated model** and the deferred-face limit of 5000 are both marked
for tuning in the source. Exceeding the first is a fatal load error
([`model.c`](model.c.md#mod_loadaliasmodel)); exceeding the second overruns an array.

**The liquid warp's three numbers** — an amplitude of 8 units in 16.16 fixed point, a secondary
amplitude of 3, and a speed of 20 — are the entire specification of how water surfaces ripple. A
rebuild reproducing this engine's look needs exactly these.

## Reporting variables

```text
VARIABLE r_outofsurfaces, r_outofedges : int      # how many were dropped
VARIABLE r_maxsurfsseen, r_maxedgesseen : int     # the high-water marks
VARIABLE r_cnumsurfs : int ;  r_surfsonstack : bool
VARIABLE r_polycount, r_wholepolycount, r_drawnpolycount, c_faceclip : int
VARIABLE r_amodels_drawn : int
VARIABLE r_time1, dp_time1, dp_time2, db_time1, db_time2, rw_time1, rw_time2,
         se_time1, se_time2, de_time1, de_time2, dv_time1, dv_time2 : real
```

**Invariants** — the renderer **counts what it had to drop** and reports it on request, and it tracks
high-water marks so a level's real requirements can be measured. That instrumentation is why the pool
sizes above could be tuned at all, and a rebuild should keep it: a renderer that silently degrades
under load is a renderer nobody can size.

The twelve timing variables bracket six passes — polygon drawing, surface drawing, world rendering,
edge scanning, edge drawing and view drawing — and are printed by a console command. The engine's own
profiler.

## Tunable variables

Eighteen console variables, of which the ones that change behaviour rather than reporting are: the
draw order, the ambient light floor, the full-bright override, whether entities are drawn at all, the
clear colour, the underwater warp toggle, and the surface and edge pool maxima.

**Invariants** — the ambient floor and the full-bright override are the two that visibly change the
world's lighting and are the standard way this engine is made playable in a dark room.

## Entry points

**Contract** — the rest of the file declares the software renderer's internal surface, grouped by
implementing file: the world walk and face rendering
([`r_bsp.c`](r_bsp.c.md), [`r_surf.c`](r_surf.c.md)), the edge emission and scan
([`r_edge.c`](r_edge.c.md)), the animated-model transform and clip
([`r_alias.c`](r_alias.c.md), [`r_aclip.c`](r_aclip.c.md)), sprites
([`r_sprite.c`](r_sprite.c.md)), particles ([`r_part.c`](r_part.c.md)), lighting
([`r_light.c`](r_light.c.md)), the sky ([`r_sky.c`](r_sky.c.md)), entity fragments
([`r_efrag.c`](r_efrag.c.md)), the frame setup and reporting
([`r_main.c`](r_main.c.md), [`r_misc.c`](r_misc.c.md)), and the surface-block
rasterizers, which exist in a portable form and in four mip-specific assembly forms selected at
compile time.

**Invariants** — the four mip-specific surface-block routines are the clearest example of the
assembly seam in this engine: the portable code has one routine parameterized by mip level, and the
assembly has four specialized ones. The contract is identical
([Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)).

The six start-and-end pairs bracket runs of assembly that need the floating-point control word set
([`sys.h`](sys.h.md)), and are empty in the portable build.
