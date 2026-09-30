# WinQuake/d_iface.h

> The rasterizer interface: the primitives the renderer hands down, the capability flags by which a rasterizer states what shape of input it wants, and the surface-generation callback that goes back up.

**Needs** — [`model.h`](model.h.md) · [`render.h`](render.h.md) · [`vid.h`](vid.h.md) · [`mathlib.h`](mathlib.h.md) · [`cvar.h`](cvar.h.md)
**Used by** — every `d_*.c` file implements it; [`r_main.c`](r_main.c.md), [`r_edge.c`](r_edge.c.md), [`r_alias.c`](r_alias.c.md), [`r_sprite.c`](r_sprite.c.md), [`r_part.c`](r_part.c.md) and [`r_surf.c`](r_surf.c.md) call it; [`d_ifacea.h`](d_ifacea.h.md) mirrors several of its records for the assembly
**Tier floor** — T1 as written; the fixed-point vertex format and the shared descriptor globals are what bind it

## Purpose

The engine is split into a **renderer** (the `r_*` files: decides what is visible, transforms it,
sorts it) and a **rasterizer** (the `d_*` files: turns primitives into pixels). This file is the
boundary, and it is the only place in the engine that was designed as a *pluggable* interface — the
capability flags below exist so that a rasterizer with different needs could be substituted, and the
hardware renderer is exactly that substitution.

Two things are worth carrying out of it.

**Primitives are passed through shared descriptor globals, not arguments.** The renderer fills in a
record and calls a parameterless function. That is a 1996 calling-convention saving and is purely
incidental — except that it makes the interface non-reentrant, which is worth knowing.

**The capability flags are a real negotiation.** A rasterizer declares whether it wants clipped
polygons or a span list, whether it wants them front-to-back or back-to-front, whether it can use
recursive subdivision, and how many bytes a pixel is. The renderer reads those and changes what it
does. A rebuild that supports one rasterizer can delete them; a rebuild supporting both a software
and a hardware path needs them or their equivalent.

## State

```text
CONSTANT warp_width  = 320     # the sky/water warp buffer's dimensions
CONSTANT warp_height = 200
CONSTANT max_lbm_height = 480  # the tallest image any loader will accept
CONSTANT particle_z_clip = 8.0 # particles nearer than this are dropped
CONSTANT turb_tex_size = 64    # the liquid warp's base texture size
CONSTANT cycle         = 128   # the liquid warp's cycle length
CONSTANT tile_size     = 128   # the size of a generated tiled surface
CONSTANT skyshift      = 7     # so the sky layers are 128 by 128
CONSTANT transparent_color = 0xFF   # the palette index meaning "skip"
```

**Invariants** — palette index 255 is transparent, everywhere, and the constant is duplicated in the
assembly's own header. The liquid warp's cycle of 128 and texture size of 64 are what make the sine
table's indexing a mask rather than a modulo.

## The primitives

```text
RECORD EmitPoint                # a transformed, projected point
  u, v : real                   # screen position
  s, t : real                   # texture coordinate
  zi   : real                   # ONE OVER depth — see the note

RECORD PolyVert                 # a clipped polygon's vertex
  u, v, zi, s, t : real

RECORD PolyDesc                 # one clipped polygon
  numverts     : int
  nearzi       : real           # the largest 1/z, i.e. the NEAREST point
  pcurrentface : MSurface
  pverts       : list<PolyVert>

RECORD FinalVert                # an animated model's vertex, ready to rasterize
  v     : int[6]                # u, v, s, t, light, 1/z — ALL FIXED POINT
  flags : int                   # which clip planes it is outside
  reserved : real

RECORD AffineTriDesc            # one animated model's mesh
  pskin        : bytes
  pskindesc    : MAliasSkinDesc
  skinwidth, skinheight : int
  ptriangles   : list<MTriangle>
  pfinalverts  : list<FinalVert>
  numtriangles : int
  drawtype     : int
  seamfixupX16 : int            # half the skin width, in 16.16 fixed point

RECORD ScreenPart               # a projected particle
  u, v, zi, color : real

RECORD SpriteDesc               # one sprite quad
  nump         : int
  pverts       : list<EmitPoint>   # with room for one extra, so a rasterizer
                                   # may duplicate vertex 0 at the end and
                                   # avoid wrapping
  pspriteframe : MSpriteFrame
  vup, vright, vpn : vec3          # in world space
  nearzi       : real

RECORD ZPointDesc               # a single depth-buffered point
  u, v  : int
  zi    : real
  color : int

RECORD Particle                 # a live particle
  org   : vec3                  # the rasterizer may read these two
  color : real
  next  : Particle              # the rasterizer never touches the rest
  vel   : vec3
  ramp  : real                  # position along its colour ramp
  die   : real                  # the time it expires
  type  : enum { static, grav, slowgrav, fire, explode, explode2, blob, blob2 }
```

**Invariants** — **depth is carried as its reciprocal throughout.** Every vertex has `zi`, meaning
one over the view-space depth, and every comparison is "larger is nearer". That is because
perspective-correct interpolation is linear in one over depth, so the reciprocal can be interpolated
across a span while the depth itself cannot. A rebuild must keep the reciprocal form in the
rasterizer even if it uses depth elsewhere.

**An animated model's vertices are handed over as integers**, in fixed point, including the light
value. So the transform, the projection and the lighting are all done by the renderer and the
rasterizer only interpolates. That is what makes the triangle rasterizer as simple as it is.

The **seam fix-up value** is half the skin width in 16.16 fixed point, precomputed here so the
back-facing-triangle adjustment ([`modelgen.h`](modelgen.h.md)) is one addition.

The particle record's comment marks the split: the first two fields are the rasterizer's input and
the rest are the simulation's private state. A rebuild should separate them into two records.

## The capability flags

```text
VARIABLE r_pixbytes : int                # 1 or 2 bytes per destination pixel
VARIABLE r_drawpolys : bool              # wants CLIPPED POLYGONS, not a span list
VARIABLE r_drawculledpolys : bool        # wants polygons already culled by the
                                         # edge list
VARIABLE r_worldpolysbacktofront : bool  # wants them back to front
VARIABLE r_recursiveaffinetriangles : bool  # can use recursive subdivision past
                                            # a distance
VARIABLE r_aliasuvscale : real           # the sub-pixel scale the renderer should
                                         # apply to model vertices
VARIABLE d_con_indirect : int            # 0: the engine draws the console straight
                                         # into the surface; 1: it must go through
                                         # the rasterizer
VARIABLE r_dowarp : bool                 # the frame must be rendered into the warp
                                         # buffer and warped on the way out
```

**Invariants** — these are set by the rasterizer at initialization and read by the renderer every
frame. The four polygon flags describe genuinely different architectures: the shipping software
rasterizer wants a **span list**, sorted front to back, produced by the edge sorter; a hypothetical
hardware one would want clipped polygons back to front and would use a depth buffer instead. The
software path sets all four to false and the code paths for the alternatives exist but are unexercised
in this build.

**The pixel width is a *rasterizer* property that reaches all the way into the model and sprite
loaders** ([`model.c`](model.c.md#mod_loadaliasskin-mod_loadaliasskingroup)), which expand skins through the palette at load
time. So changing it requires reloading every asset — which is why a resolution change reloads the
level.

The console-indirection flag exists because some backends cannot expose a writable surface for the
console's own drawing.

## The calls down

**Contract** — `D_Init` sets up the rasterizer and declares its capabilities. `D_ViewChanged` and
`D_SetupFrame` respond to a view change and begin a frame. `D_DrawSurfaces` consumes the span list.
`D_DrawPoly`, `D_DrawSprite`, `D_DrawZPoint` and `D_PolysetDraw` each rasterize the primitive in the
matching descriptor global. `D_PolysetDrawFinalVerts` draws a run of model vertices as points, for the
recursive-subdivision path. `D_StartParticles`, `D_DrawParticle` and `D_EndParticles` bracket the
particle pass. `D_WarpScreen` applies the underwater warp. `D_FillRect`, `D_DrawRect` and
`D_UpdateRects` are the two-dimensional operations. `D_BeginDirectRect` and `D_EndDirectRect` draw
straight to the display, bypassing the surface, for the disc-access indicator.
`D_EnableBackBufferAccess` and `D_DisableBackBufferAccess` bracket access to the drawing surface on
backends that need it.

**Invariants** — the direct-rectangle pair is how the disc indicator appears *during* a level load,
when the normal frame loop is not running ([`common.c`](common.c.md#com_loadfile-and-its-four-wrappers)). It writes to the
display rather than to the off-screen surface, which is why it needs its own path.

The back-buffer bracket is the same locking concern as
[`vid.h`](vid.h.md#vid_lockbuffer-vid_unlockbuffer), at a different level.

## The call up

```text
RECORD DrawSurf                 # the surface-generation request
  surfdat   : pixel[]           # where to write the generated surface
  rowbytes  : int
  surf      : MSurface          # what to generate
  lightadj  : fixed8[4]         # the four light styles' current levels,
                                # including dynamic lights
  texture   : Texture           # already resolved through the animation cycle
  surfmip   : int               # texels per world pixel, as a power of two
  surfwidth, surfheight : int   # in mipped texels
```

**Contract** — `R_DrawSurface` generates one lit, mipped surface into the request's destination.
`R_GenTile` generates a tiled surface for an unlit one.

**Invariants** — this is the **callback direction**: the rasterizer's surface cache discovers it needs
a surface and calls back up into the renderer to build it
([`d_surf.c`](d_surf.c.md) calls [`r_surf.c`](r_surf.c.md)). That inversion is what makes the cache
demand-driven, and it is the reason the two halves are not cleanly layered.

The light adjustment is **fixed point with eight fractional bits** and already combines the static
light styles with any dynamic lights touching the surface — so the surface builder does no lighting
decisions, only arithmetic.

The texture has already been resolved through its animation cycle
([`model.c`](model.c.md#mod_loadtextures)), so the builder never sees the cycle.

## Sky and warp state

```text
VARIABLE r_skydirect : int ;  r_skysource : bytes
VARIABLE skyspeed, skyspeed2, skytime : real
VARIABLE r_warpbuffer : bytes
VARIABLE acolormap : pointer       # the current shading table
VARIABLE d_spanpixcount, c_surf : int    # counters
VARIABLE r_framecount : int              # frames since startup
VARIABLE scr_vrect : Rect
```

**Invariants** — the two sky speeds are the two layers' scroll rates, and the difference between them
is what makes the sky look like clouds over a background rather than one scrolling image.

The warp buffer is where a frame is rendered when the underwater warp is active; the frame is then
resampled out of it ([`d_scan.c`](d_scan.c.md#d_warpscreen)). Its size is fixed at 320 by 200
regardless of the display's resolution, which is why the underwater effect is blockier at high
resolutions.

**Notes** — the file's own comments mark several of these as internal, as belonging elsewhere, or as
things that should go away. The shading-table global in particular is flagged for removal and is
read by the assembly. A rebuild should pass it.
