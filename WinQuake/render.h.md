# WinQuake/render.h

> The renderer's public interface: the per-frame view description, the client-side entity, the entity-to-leaf fragment list, and the particle-effect calls.

**Needs** — [`vid.h`](vid.h.md) (the rectangle type) · [`mathlib.h`](mathlib.h.md) · [`quakedef.h`](quakedef.h.md) (the network entity state)
**Used by** — [`r_main.c`](r_main.c.md) and [`gl_rmain.c`](gl_rmain.c.md) implement it; [`client.h`](client.h.md) embeds the entity type; [`view.c`](view.c.md), [`screen.c`](screen.c.md), [`cl_main.c`](cl_main.c.md), [`cl_tent.c`](cl_tent.c.md) and [`r_part.c`](r_part.c.md) call it
**Tier floor** — none for the interface; the view description's precomputed fields are T1 artifacts

## Purpose

Two renderers implement this file — the software one and the hardware one — and the client talks to
whichever is compiled in through exactly these calls. That makes it the seam between the client and
the renderer, and it is worth reading for what the client *must* provide: a view description, a list
of entities, and nothing else.

The view description is the interesting part. It contains the view position and angles that a caller
sets, and then about twenty **precomputed derivatives** that the renderer fills in when the view
changes — clamping edges, shifted fixed-point bounds, floating-point copies of integer fields. Every
one exists so an inner loop can avoid a conversion or a comparison. A rebuild computes almost none of
them and keeps the six fields that are actually inputs.

## State

```text
CONSTANT maxclipplanes = 11    # the view frustum plus the sub-model's own planes
CONSTANT top_range     = 16    # the palette row where shirt colours begin
CONSTANT bottom_range  = 96    # where trouser colours begin

RECORD EFrag                    # one entity's presence in one leaf
  leaf     : MLeaf
  leafnext : EFrag              # the next entity in this leaf
  entity   : Entity
  entnext  : EFrag              # the next leaf this entity touches

RECORD Entity                   # the CLIENT's view of a thing to draw
  forcelink   : bool            # the model changed: do not interpolate
  update_type : int
  baseline    : EntityState     # what the server said at spawn; defaults for
                                # updates
  msgtime     : real            # when the last update arrived
  msg_origins : vec3[2]         # the last two positions, newest first
  origin      : vec3            # the INTERPOLATED position
  msg_angles  : vec3[2]         # the last two angle triples, newest first
  angles      : vec3            # the interpolated angles
  model       : optional<Model>
  efrag       : EFrag           # the leaves this entity touches
  frame       : int
  syncbase    : real            # the animation's phase offset
  colormap    : bytes           # the palette translation for a player
  effects     : int             # the entity effect bits of server.h
  skinnum     : int
  visframe    : int             # the frame this was last found visible
  dlightframe : int             # the frame a dynamic light last touched it
  dlightbits  : int
  trivial_accept : int          # the culling result: fully inside the frustum
  topnode     : MNode           # for a sub-model, the first world node that
                                # splits it, or nothing
```

**Invariants** — the entity holds **the last two updates and an interpolated value** for both
position and angles, which is the whole of client-side smoothing
([`cl_main.c`](cl_main.c.md#cl_relinkentities)). The server sends at its tick rate and the client
draws at its frame rate, so every drawn position is a blend of two received ones.

The `forcelink` flag suppresses interpolation for one frame, and it is set when the model changed or
when the server marked the update as a teleport ([`protocol.h`](protocol.h.md)). Without it a
teleporting entity streaks across the map.

The entity-fragment list is a **two-way cross-reference**: each fragment belongs to one leaf and one
entity, and each is in two singly linked lists. That is how the renderer answers "which entities are
in this visible leaf" without searching, and it is rebuilt whenever an entity moves
([`r_efrag.c`](r_efrag.c.md)).

The colour map is a 256-byte palette translation built per player from their shirt and trouser
colours, remapping the two palette ranges named by the constants above. A non-player entity has
none.

## The view description

```text
RECORD RefDef
  # --- inputs a caller sets ---
  vieworg    : vec3
  viewangles : vec3
  fov_x, fov_y : real
  ambientlight : int
  vrect      : Rect             # the sub-window of the display to draw into

  # --- derived when the view changes; see the note ---
  aliasvrect : Rect             # the same window, scaled for animated models
  vrectright, vrectbottom             : int
  aliasvrectright, aliasvrectbottom   : int
  vrectrightedge                      : real
  fvrectx, fvrecty                    : real
  fvrectx_adj, fvrecty_adj            : real   # clamping edges
  vrect_x_adj_shift20                 : int    # (x + 0.5 - epsilon) << 20
  vrectright_adj_shift20              : int
  fvrectright_adj, fvrectbottom_adj   : real
  fvrectright, fvrectbottom           : real
  horizontalFieldOfView               : real   # visible width at unit depth;
                                               # 2.0 is 90 degrees
  xOrigin, yOrigin                    : real   # the projection centre, as a
                                               # fraction: x is always 0.5,
                                               # y is 0.3 to 0.5
```

**Invariants** — only the six inputs matter to a rebuild. Everything else is
[`r_main.c`](r_main.c.md#r_viewchanged) precomputing: integer bounds as floats to avoid conversions
in comparisons, bounds biased by half a pixel and an epsilon to make clamping a single test, and two
of them pre-shifted into 20-bit fixed point because the edge rasterizer works in that format.

The **vertical projection centre is not 0.5.** It sits between 0.3 and 0.5, which raises the
horizon: the player looks slightly downward at their own feet. That is a deliberate composition
choice and it is visible in every screenshot of this game.

The field of view is expressed as **visible width at unit depth**, not as an angle, because that is
the form the projection needs.

The scaled rectangle for animated models exists because the software renderer rasterizes models at a
different sub-pixel scale than surfaces ([`r_alias.c`](r_alias.c.md)).

## Globals

```text
VARIABLE r_refdef        : RefDef
VARIABLE r_origin        : vec3          # the view position
VARIABLE vpn, vright, vup : vec3         # the view basis
VARIABLE r_notexture_mip : Texture       # the checkerboard placeholder
VARIABLE reinit_surfcache : int          # the surface cache is empty
VARIABLE r_cache_thrash  : bool          # the surface cache is thrashing
```

**Invariants** — the view basis is a global rather than part of the view description, and the sound
system reads it ([`host.c`](host.c.md)) to spatialize. So the renderer's basis is also the
listener's.

The thrashing flag is set when the surface cache evicted something it needed again in the same frame,
and the renderer *tells the player* by flashing the screen edge
([`screen.c`](screen.c.md)). A rebuild should keep the signal; a silently thrashing cache is
invisible and catastrophic for frame time.

## Lifecycle

**Contract** — `R_Init` sets up the renderer. `R_InitTextures` builds the placeholder texture and is
called **even on a dedicated server**, because the map loader needs it. `R_InitEfrags` clears the
fragment pool. `R_NewMap` prepares for a new level. `R_InitSky` builds the sky's two-layer texture
from a loaded texture, at level load.

## Per frame

**Contract** — `R_RenderView` draws the world; the caller must have filled in the view description
first. `R_ViewChanged` recomputes every derived field and must be called whenever the view
description or the display's shape changes. `R_PushDlights` marks the surfaces every dynamic light
touches, before drawing. `R_SetVrect` computes the drawing rectangle from the display's size and a
number of lines to reserve at the bottom for the status bar.

**Invariants** — the separation between setting the view and recomputing derivatives is what makes
the twenty precomputed fields possible: they are computed once per view change, not once per frame.

## Entity fragments

**Contract** — `R_AddEfrags` walks the world tree inserting an entity into every leaf it touches;
`R_RemoveEfrags` unlinks it from all of them. Called whenever an entity moves.

## Particle effects

**Contract** — nine effect constructors, each spawning a burst with its own colour ramp, velocity
distribution and lifetime: a parsed network effect, a generic burst, a rocket or gib trail, an
entity's sparkle field, the tarbaby and plain explosions, a coloured explosion with a palette range,
the lava splash and the teleport flash.

**Invariants** — these are **client-side only**: the server sends an event and the client invents the
particles ([`protocol.h`](protocol.h.md)). So two clients watching the same explosion see different
particles, and a recorded demo replays the event rather than the particles.

## Surface cache

**Contract** — `D_SurfaceCacheForRes` returns the bytes a given resolution needs, so the video
backend can reserve them. `D_InitCaches` establishes the cache in a caller-supplied buffer.
`D_FlushCaches` empties it. `D_DeleteSurfaceCache` releases it.

**Invariants** — these five declarations sit in the renderer's *public* header rather than in the
software renderer's own, because the **video backend** calls them: it decides the resolution, so it
must size and allocate the cache ([`vid_win.c`](vid_win.c.md) and its siblings). The hardware
renderer implements them as stubs. A rebuild should keep the inversion — whoever owns the surface
must size the cache.
