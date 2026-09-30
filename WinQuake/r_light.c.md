# WinQuake/r_light.c

> Light styles, dynamic-light marking, and the point light sample — the three pieces of lighting that are not in the surface builder.

**Needs** — [`r_local.h`](r_local.h.md) · [`model.h`](model.h.md) · [`client.h`](client.h.md) · [`mathlib.h`](mathlib.h.md)
**Used by** — [`cl_main.c`](cl_main.c.md) animates styles and pushes lights; [`r_surf.c`](r_surf.c.md) reads the style levels and the light bits; [`r_main.c`](r_main.c.md) samples the point light for models
**Tier floor** — none

## Purpose

The three pieces of lighting the surface builder does not own: advancing the light styles, deciding which surfaces each
dynamic light touches, and sampling the baked lightmap under a point. Read
[`r_surf.c`](r_surf.c.md) for what consumes the first two and [`r_main.c`](r_main.c.md) for what consumes the third.

## State

```text
VARIABLE d_lightstylevalue[styles]    # each style's current level, as a fraction
VARIABLE r_dlightframecount           # which frame the dynamic lights were marked in
VARIABLE lightspot                    # where the last point sample hit (unused here)
```

**Invariants** — a style's level is a fixed-point fraction, not a floating value, because the surface builder multiplies
stored samples by it in integer arithmetic ([`r_surf.c`](r_surf.c.md)).

## `R_AnimateLight`

**Contract** — converts every light style's animation string into a current level, once per frame, from the
clock. A style with no string is fully lit.

```text
FUNCTION r_animate_light()
  i = truncate(cl.time * 10)                   # ten steps per second
  FOR EACH style j IN 0..63
    IF the style has no string
      d_lightstylevalue[j] = 256 ;  CONTINUE   # 1.0 in 8.8 fixed point
    k = i MODULO the string's length           # cycle
    k = the letter at position k, minus 'a'    # 'a' = 0 .. 'z' = 25
    d_lightstylevalue[j] = k * 22
```

**Invariants** — the letter-to-level mapping is `(letter - 'a') * 22`, so `a` is 0, `m` is 264 and `z` is 550.
The comment states the convention: **`m` is normal, `a` is dark, `z` is double bright.** And `m` at 264 is
slightly *above* the unlit default of 256 — so a style string of all `m` is marginally brighter than no
string at all, which is a real and visible inconsistency in shipped maps.

The factor 22 is chosen so that 26 letters span roughly 0 to 2.15 in 8.8 fixed point. The levels feed the
surface builder ([`r_surf.c`](r_surf.c.md#r_buildlightmap)) as per-layer scales.

**Ten steps per second**, so a style string of length 10 cycles once per second.

## `R_MarkLights`, `R_PushDlights`

**Contract** — `R_PushDlights` walks the world tree once per live dynamic light, marking every surface within
that light's radius. `R_MarkLights` is the recursion: it descends only into the half-spaces the light's sphere
reaches, and where the sphere straddles a plane, marks that node's faces and descends both ways.

```text
FUNCTION r_push_dlights()
  r_dlightframecount = r_framecount + 1        # the frame counter has not
                                               # advanced yet
  FOR EACH light i
    IF it has expired OR has no radius  CONTINUE
    r_mark_lights(light, bit i, the world's root)

FUNCTION r_mark_lights(light, bit, node)
  IF node IS a leaf  RETURN
  dist = the signed distance FROM the light TO node's plane
  IF dist >  light.radius  recurse INTO the front child ONLY ;  RETURN
  IF dist < -light.radius  recurse INTO the back child ONLY ;  RETURN
  # The sphere straddles: this node's faces are in reach.
  FOR EACH face OF this node
    IF face.dlightframe != r_dlightframecount        # first light this frame
      face.dlightbits = 0 ;  face.dlightframe = r_dlightframecount
    face.dlightbits = face.dlightbits BITOR bit
  recurse INTO BOTH children
```

**Invariants** — four things.

**The frame stamp is the *next* frame's count**, because the marking runs before the frame counter advances.
That off-by-one is deliberate and the comment says so; a rebuild that advances the counter first must drop it.

**The bit set is cleared lazily**, on the first light to touch a surface in a given frame, rather than by a
clearing pass over every surface. That is what makes the marking proportional to the lights rather than to the
map.

**A light's reach is tested against a plane, not a box**, so the descent is conservative: a node whose plane
the sphere reaches has all its faces marked even if the sphere is nowhere near them. The surface builder's
per-sample distance test ([`r_surf.c`](r_surf.c.md#r_adddynamiclights)) is what actually bounds the
contribution.

**A marked surface is rebuilt every frame** ([`d_surf.c`](d_surf.c.md#d_cachesurface)), so this marking
directly sets the frame's surface-cache workload. A rocket in a corridor marks dozens of surfaces and rebuilds
all of them, sixty times a second.

## `RecursiveLightPoint`, `R_LightPoint`

**Contract** — `R_LightPoint` samples the world's baked lighting at a point by tracing downward and reading the
lightmap of the first lit surface hit. Returns a light level, or zero when the map has no lighting data.
`RecursiveLightPoint` is the trace.

```text
FUNCTION r_light_point(p) -> int
  IF the world has no lighting data  RETURN 0
  end = p, lowered by 2048
  r = recursive_light_point(the world's root, p, end)
  RETURN 0 IF r == -1 ELSE r

FUNCTION recursive_light_point(node, start, end) -> int   # -1 means nothing hit
  IF node IS a leaf  RETURN -1
  front = the signed distance FROM start TO the plane
  back  = the same FOR end
  side = (front < 0)
  IF both endpoints are on the same side
    RETURN recursive_light_point(node.children[side], start, end)

  mid = the segment's intersection with the plane
  r = recursive_light_point(node.children[side], start, mid)    # NEAR side
  IF r >= 0  RETURN r                                          # hit something

  FOR EACH face OF this node
    IF it has the tiled flag  CONTINUE                # no lightmap
    s, t = the intersection's texture coordinates
    IF (s,t) is outside the face's extents  CONTINUE
    IF the face has no lightmap samples  RETURN 0     # unlit, but it IS a hit
    # Read the lightmap sample at (s,t)/16, summing every light style's layer
    # scaled by its current level.
    total = 0
    FOR EACH style layer
      total = total + sample * d_lightstylevalue[style] / 256
    RETURN total

  RETURN recursive_light_point(node.children[NOT side], mid, end)  # FAR side
```

**Invariants** — four things.

**The trace is 2048 units straight down**, so an entity more than 2048 units above any lit surface is
unlit — which reads as a monster going black in a very tall room.

**A single lightmap sample is read, not interpolated**, so a model's brightness changes in 16-unit steps as it
moves. Visible on a large moving platform.

**Surfaces with no lightmap return zero rather than continuing**, treating an unlit surface as a hit. So a
model standing on a sky or liquid surface is unlit rather than sampling the floor below it.

**The sample sums every light style's layer scaled by its current level**, the same combination the surface
builder performs — which is what makes a model under a flickering light flicker with it.

This is the *only* source of positional lighting for models
([`r_main.c`](r_main.c.md#r_drawentitiesonlist) uses a fixed light *direction*), so a model's overall
brightness tracks the room while its shading direction does not.
