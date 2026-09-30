# WinQuake/gl_rlight.c

> Light styles, dynamic light marking and the point-light lookup, plus the hardware renderer's cheap alternative to real dynamic lights: a blended blob drawn in front of the camera.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`gl_model.h`](gl_model.h.md)
**Used by** — [`gl_rmain.c`](gl_rmain.c.md) · [`gl_rsurf.c`](gl_rsurf.c.md) · [`r_part.c`](r_part.c.md)
**Tier floor** — none

## Purpose

Read [`r_light.c`](r_light.c.md) for the shared content: the light-style strings and how a letter becomes an intensity,
the marking of which surfaces each dynamic light touches, and the recursive trace down the world tree that finds the
lightmap sample under a point. This twin records the two additions.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## What differs

**A dynamic light can be drawn as a blob instead of lighting surfaces.** With the blob switch on, the surface-marking
pass is skipped entirely and each light is instead drawn as a small blended fan of vertices facing the viewer, with a
bright centre and a dark rim.

```text
FUNCTION render_dlight(light)
  rad = light.radius * 0.35
  v = light origin - view origin
  IF |v| < rad  the viewer is INSIDE the light
    add a full-screen tint proportional to the radius ;  RETURN
  emit a fan: centre at origin + 0.25*rad along the view direction, bright
              then 16 rim vertices on a circle of radius rad, black
```

**Invariants** —

- **It costs one blended fan per light instead of rebuilding every lightmap rectangle the light touches.** That is the
  trade the switch offers, and on the hardware of the day it was often the only way to keep the frame rate with several
  rockets in flight.
- The blob is drawn **slightly toward the viewer** from the light's true position, so it is not buried in the wall the
  rocket is about to hit.
- A viewer **inside** the light's radius cannot be given a blob — the fan would be behind them — so the light becomes a
  full-screen tint instead, added to the same blend the damage flash uses ([`view.c`](view.c.md)). Without this case a
  rocket passing through the player's own position simply vanishes.
- The blobs are drawn with depth **testing** on but no depth write, so they are occluded by walls but do not occlude each
  other. They still show through nothing, which is why a rocket behind a corner does not light it — the visible
  difference from real dynamic lights, and the reason the switch defaults off.

**The point-light trace records where it hit.** The recursive descent that samples a lightmap under a point also stores
the world position of the hit, which is what [`gl_rmain.c`](gl_rmain.c.md#gl_drawaliasshadow) projects the model's
shadow onto. The software renderer draws no shadows and does not need it.

**Notes** — the blob technique is worth keeping in mind even for a rebuild that does not need it: it is the general
answer to "a light source the player should see even when its illumination is too expensive", and it is why muzzle
flashes in this era of games look like glowing spheres.
