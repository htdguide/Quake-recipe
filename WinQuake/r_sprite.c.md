# WinQuake/r_sprite.c

> Builds a sprite's quad according to one of five orientation rules, clips it against the frustum, and hands it to the rasterizer.

**Needs** — [`r_local.h`](r_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [`spritegn.h`](spritegn.h.md) · [`client.h`](client.h.md) · [`mathlib.h`](mathlib.h.md)
**Used by** — [`r_main.c`](r_main.c.md); [`d_sprite.c`](d_sprite.c.md) rasterizes
**Tier floor** — none

## Purpose

The five orientation rules of [`spritegn.h`](spritegn.h.md) are implemented here, and each is a different basis
construction. That is the file's content: everything else is clipping and dispatch.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## `R_RotateSprite`

**Contract** — for the oriented rules, rotates the sprite's basis by the entity's own angles and offsets the
view position along the sprite's normal by the beam length.

## `R_GetSpriteframe`

**Contract** — takes a sprite; returns the frame for the entity's frame number, resolving a frame group by its
cumulative intervals and the clock offset by the entity's animation phase. An out-of-range frame number is
clamped with a diagnostic.

**Invariants** — the same phase-offset mechanism as animated models
([`r_alias.c`](r_alias.c.md)), so a row of flames burns out of step.

## The five orientation rules

**Contract** — `R_SetupAndDrawSprite` builds the sprite's two basis vectors — the directions its width and
height run along — according to its type, then forms four world-space vertices from the frame's four edge
distances.

```text
FUNCTION build_sprite_basis(type) -> (up, right)
  SELECT type
    vp_parallel
      # A plain billboard: the view's own basis. Tilts with the camera.
      up = vup ;  right = vright
    vp_parallel_upright
      # Faces the view PLANE but stays vertical: world up, and a right vector
      # perpendicular to both world up and the view direction.
      up = (0, 0, 1)
      right = normalize(cross(the view direction, up))
    facing_upright
      # Faces the view POINT rather than the plane: right is perpendicular to
      # world up and the direction FROM the viewer TO the sprite.
      up = (0, 0, 1)
      right = normalize(cross(the entity's origin - r_origin, up))
    oriented
      # Fixed in the world by the entity's own angles: no view dependence.
      up, right = from the entity's angle triple
    vp_parallel_oriented
      # A billboard, then rolled by the entity's own roll angle.
      angle = the entity's roll
      up, right = the view basis ROTATED by that angle in the view plane
```

**Invariants** — the differences are exactly as [`spritegn.h`](spritegn.h.md) describes and each exists for a
specific effect: a tall flame must not pitch with the camera, an explosion must, the lightning bolt must not
turn at all, and a spinning effect needs its own roll. A rebuild implementing only the plain billboard gets
most sprites subtly and some grossly wrong.

## `R_ClipSpriteFace`

**Contract** — clips the sprite's polygon against one frustum plane, generating vertices where it crosses;
returns the resulting vertex count.

**Invariants** — a quad clipped against four planes can reach eight vertices, and the working array is sized
for it. A count below three rejects the sprite.

## `R_DrawSprite`

**Contract** — selects the frame, builds the basis, forms the quad from the frame's edge distances, clips it
against every frustum plane the entity might straddle, projects the survivors, fills the sprite descriptor and
calls the rasterizer.

**Invariants** — the frame's four **signed edge distances**
([`model.h`](model.h.md)) are what position the quad relative to the entity, which is why the loader converts
the file's pixel origin into them.

**Notes** — the source notes that the clipping-plane selection should be caller-selectable. A rebuild passes
the clip flags in, as the face renderer does.
