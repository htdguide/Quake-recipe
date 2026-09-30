# WinQuake/r_shared.h

> The two records the whole software rasterizer turns on — an active edge and a surface being scanned — plus the span list and the capacities that bound a frame.

**Needs** — [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [`render.h`](render.h.md) · [`mathlib.h`](mathlib.h.md) · [`cvar.h`](cvar.h.md)
**Used by** — [`r_local.h`](r_local.h.md) includes it; every `r_*.c` and `d_*.c` file reads it; [`asm_draw.h`](asm_draw.h.md) and [`d_ifacea.h`](d_ifacea.h.md) mirror its record layouts for the assembly
**Tier floor** — T1 as written: the records are padded to cache-line boundaries and their field offsets are duplicated in assembly

## Purpose

Two records here carry the software renderer's whole architecture, and understanding them is
understanding how this engine draws a world without a depth buffer.

An **edge** is one screen-space line segment of a polygon, with a fixed-point x coordinate and a
per-scanline step. A **surface** is a polygon currently being scanned, holding the depth gradient that
lets the sorter decide which of two overlapping surfaces is nearer at a given pixel. The renderer
emits every visible polygon's edges into a per-scanline bucket list; the sorter walks the screen one
scanline at a time, maintaining a sorted active edge list and a depth-sorted active surface stack, and
emits a **span** — a horizontal run of pixels belonging to exactly one surface — whenever the topmost
surface changes.

That is Carmack's edge-sorted span renderer, and its consequence is the engine's defining
performance property: **every pixel of the world is written exactly once**, with no overdraw and no
depth test. A rebuild targeting modern hardware will almost certainly use a depth buffer instead and
should — but it must then understand that the surface cache, the front-to-back tree walk and the
mip selection all exist to serve this architecture and can be simplified along with it.

## State

```text
CONSTANT maxverts        = 16      # points in one surface polygon
CONSTANT maxworkingverts = 20      # while clipping
CONSTANT maxheight       = 1024    # the tallest display supported
CONSTANT maxwidth        = 1280
CONSTANT sin_buffer_size = maxdimension + 128
CONSTANT infinite_distance = 0x10000   # farther than anything in a scene

CONSTANT numstackedges    = 2400   # edges in a frame before the pool is grown
CONSTANT numstacksurfaces = 800
CONSTANT maxspans         = 3000   # spans emitted per scanline batch
```

**Invariants** — the vertex limit of 16 per polygon is what the map compiler's face subdivision
respects. The display limits bound the per-scanline bucket arrays, which are static.

The stack capacities are the *initial* pools, allocated on the stack when they fit; a level needing
more gets heap pools ([`r_main.c`](r_main.c.md)). Exceeding them mid-frame drops the excess and sets a
reported counter — the frame renders incompletely rather than failing, which is the correct behaviour
for a real-time renderer and is worth preserving.

## The span

```text
RECORD ESpan                    # one horizontal run of one surface
  u, v, count : int             # start x, scanline y, length in pixels
  pnext       : ESpan
```

**Invariants** — a span belongs to exactly one surface and one scanline, and the rasterizer draws it
with one inner loop. The list is per surface, so the rasterizer walks surfaces and draws each one's
spans together — which is what keeps the texture, the lightmap and the mip level constant across an
inner loop.

## The surface

```text
RECORD Surf                     # a polygon currently being scanned
  next, prev  : Surf            # the active surface stack, depth-ordered
  spans       : ESpan           # the spans accumulated for it this frame
  key         : int             # the sorting key: the tree's back-to-front order
  last_u      : int             # where its current span began
  spanstate   : int             # 0 not in a span, 1 in one, -1 in an inverted one
  flags       : int             # the surface flags of model.h
  data        : pointer         # the MSurface, or other per-kind data
  entity      : Entity
  nearzi      : real            # the nearest 1/z on it, for mip selection
  insubmodel  : bool
  d_ziorigin, d_zistepu, d_zistepv : real    # the depth GRADIENT
  pad         : int[2]          # to exactly 64 bytes
```

**Invariants** — five things here are load-bearing.

**The depth gradient is the whole reason there is no depth buffer.** A polygon is planar, so one over
its depth is a *linear* function of screen position: `zi = d_ziorigin + u*d_zistepu + v*d_zistepv`.
Comparing two surfaces at a pixel is therefore three multiplies and a compare, with no per-pixel
storage. That is the trade this architecture makes — arithmetic instead of memory — and in 1996
memory bandwidth was the scarce resource.

**The sorting key is the tree's traversal order**, and surfaces are allocated from a pool in that
order, so a **higher pool address means nearer the viewer**. The comment states it: a surf pointer
greater than another should be drawn in front. So the common case of two non-intersecting surfaces is
resolved by comparing addresses, and the gradient comparison is only needed where they interpenetrate.

**Pool entries 0 and 1 are reserved.** Entry 0 means "no surface", because an edge stores its two
surface indices as 16-bit values and zero must mean none. Entry 1 is the background — the surface
that is always behind everything — and it doubles as the active stack's sentinel. A rebuild must
reserve both or must find another way to say "none".

**The span state can be negative.** An inverted span is one whose end was reached before its start,
which happens when a surface's two edges cross in screen space after clipping. The sorter detects and
discards it rather than drawing a negative-length run.

**The record is padded to 64 bytes.** The comment says so explicitly, and it is a cache-line
alignment: the sorter touches many surfaces per scanline. Incidental to a rebuild; the padding
comment also names three fields that could be narrowed.

## The edge

```text
RECORD Edge                     # one polygon edge, active on some scanlines
  u        : fixed16            # its current x, in 16.16 fixed point
  u_step   : fixed16            # how much x changes per scanline
  prev, next : Edge             # the active edge list, sorted by u
  surfs    : int (16-bit)[2]    # the surface this edge ENTERS and the one it
                                # LEAVES; 0 means none
  nextremove : Edge             # the per-scanline removal list
  nearzi   : real
  owner    : MEdge              # the map edge it came from
```

**Invariants** — **an edge carries two surface indices, not one.** Crossing an edge left to right
*enters* one surface and *leaves* another, and that pair is what lets the sorter maintain its stack by
pushing and popping rather than re-sorting. A polygon's left edge has its surface in one slot and zero
in the other; its right edge has the reverse.

**The x coordinate is 16.16 fixed point and is stepped, not recomputed.** One addition per scanline per
active edge, which is why the whole frame's edge work is a few thousand additions. The 16 fractional
bits are the subpixel precision of every polygon boundary in the game — and the reason polygon edges
shimmer as the view moves.

**The owner pointer back into the map's edge array** is how the renderer avoids emitting the same map
edge twice when two adjacent faces share it: the map edge carries a per-frame cache offset
([`model.h`](model.h.md)) naming the emitted edge. That caching is what makes the edge count per frame
close to the number of *unique* edges rather than twice it.

## Model vertex clip flags

```text
CONSTANT alias_left_clip    = 0x0001
CONSTANT alias_top_clip     = 0x0002
CONSTANT alias_right_clip   = 0x0004
CONSTANT alias_bottom_clip  = 0x0008
CONSTANT alias_z_clip       = 0x0010
CONSTANT alias_onseam       = 0x0020   # duplicated from modelgen.h
CONSTANT alias_xy_clip_mask = 0x000F
```

**Invariants** — a model vertex's flags say which frustum planes it is outside, so a triangle whose
three vertices share no clip flag is trivially accepted and one whose flags intersect is trivially
rejected. That is the standard Sutherland trivial test, and the mask names the four screen-edge bits
so the depth bit can be handled separately — depth clipping requires actual subdivision while screen
clipping can be done by clamping.

The seam flag **shares this field** with the clip flags and must keep the value it has in the model
format. A rebuild that renumbers the clip flags must not collide with it.

## Shared globals

```text
VARIABLE cachewidth   : int ;  cacheblock : pixel[]    # the surface being sampled
VARIABLE screenwidth  : int
VARIABLE pixelAspect  : real
VARIABLE sintable, intsintable : int[]                 # the warp's sine tables
VARIABLE vup, vpn, vright : vec3                       # the view basis
VARIABLE base_vup, base_vpn, base_vright : vec3        # before a sub-model's
                                                       # rotation
VARIABLE currententity : Entity
VARIABLE surfaces, surface_p, surf_max : Surf          # the pool and its cursor
VARIABLE sxformaxis, txformaxis : vec3[4]              # the texture axes in
                                                       # view space
VARIABLE modelorg, base_modelorg : vec3                # the view position in
                                                       # model space
VARIABLE xcenter, ycenter, xscale, yscale : real       # the projection
VARIABLE xscaleinv, yscaleinv, xscaleshrink, yscaleshrink : real
VARIABLE d_lightstylevalue : int[256]                  # each style's current
                                                       # level, 8.8 fixed point
VARIABLE ubasestep, errorterm, erroradjustup, erroradjustdown : int
```

**Invariants** — the **base** copies of the view basis and the model origin exist because drawing a
rotating sub-model temporarily replaces them with rotated versions
([`r_bsp.c`](r_bsp.c.md#r_entityrotate-r_rotatebmodel)) and must restore them afterwards.

The view position **in model space** is what makes backface culling a dot product against a plane
without transforming the plane: a surface faces the viewer when the viewer is on its front side, and
that test is cheapest in the model's own space.

The light style levels are **8.8 fixed point**, so a style's brightness has eight fractional bits —
which is what lets a flickering light fade smoothly rather than in steps
([`r_light.c`](r_light.c.md)).

The four Bresenham variables are the line-scan setup's outputs, shared between the setup routine and
the assembly that consumes them.

## Entry points

**Contract** — `R_DrawLine` draws a debug line between two polygon vertices. `TransformVector` takes a
world vector into view space. `SetUpForLineScan` prepares the four Bresenham variables from two
fixed-point endpoints. `R_MakeSky` regenerates the scrolling sky texture; `r_skymade` records whether
it is current.
