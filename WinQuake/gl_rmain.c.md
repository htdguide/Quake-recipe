# WinQuake/gl_rmain.c

> The hardware frame: set the projection from the field of view, split the depth range so the world, the mirror and the view model never collide, draw the world, then entities, sprites, shadows, models and the screen tint.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`gl_model.h`](gl_model.h.md) · [`gl_rsurf.c`](gl_rsurf.c.md) · [`gl_rlight.c`](gl_rlight.c.md) · [`gl_warp.c`](gl_warp.c.md) · [`gl_mesh.c`](gl_mesh.c.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer)
**Used by** — [`gl_screen.c`](gl_screen.c.md) calls the frame entry point; [`view.c`](view.c.md) supplies the view
**Tier floor** — none

## Purpose

The hardware counterpart of [`r_main.c`](r_main.c.md). Same job — turn a view definition into a picture — and it reads
as a much shorter file, because sorting, clipping and rasterization now belong to the library. What is left is the
frame's *structure*: the order of the passes, the depth-range partitioning that stands in for having no stencil, and
the per-vertex lighting of animated models.

## State

```text
VARIABLE currententity                     # the entity being drawn
VARIABLE r_origin, vpn, vright, vup        # the view, as a point and a basis
VARIABLE frustum[4]                        # the four side planes
VARIABLE r_world_matrix, r_base_world_matrix
VARIABLE modelorg                          # the view origin in the current model's space
VARIABLE shadelight, ambientlight          # this entity's lighting, 0..255 scale
VARIABLE shadedots -> one of 16 precomputed tables indexed by vertex normal
VARIABLE shadevector                       # the direction light comes from
VARIABLE mirror, mirror_plane, mirrortexturenum
VARIABLE envmap                            # suppresses the view model and the tint
VARIABLE c_brush_polys, c_alias_polys      # counters for the speed display
```

## `R_RenderView`

**Contract** — draws one frame: clear according to the chosen scheme, set up the frame and the transforms, render the
scene, draw the deferred liquids, render the mirrored view if the map has a mirror, draw the view model, apply the
screen tint. Reports timings when asked.

```text
FUNCTION render_view()
  IF drawing is suppressed  RETURN
  IF no world is loaded  FAIL
  setup_frame()            # pick the view leaf, animate lights, count the frame
  clear()                  # also decides the depth range for this frame
  setup_transforms()
  render_scene()           # world, then entities
  draw_water_surfaces()
  draw_view_model()
  poly_blend()             # the full-screen damage/powerup tint
  IF a mirror is in view  render the mirrored scene and blend it over
```

**Invariants** — the order is forced: opaque world first so the depth buffer is populated, then blended liquids, then
the view model in its own depth slice, then the tint last because it covers everything.

## `R_Clear`

**Contract** — clears the buffers and **chooses the depth range this frame will use**, under one of three schemes: a
half-range scheme when a mirror is present, an alternating scheme that avoids clearing depth at all, or the plain full
range.

```text
FUNCTION clear()
  IF a mirror is in the map
    clear depth ;  depth range = [0, 0.5] ;  compare = nearer-or-equal
  ELSE IF the alternating trick is enabled
    do NOT clear depth
    on even frames  range = [0, 0.49999], compare = nearer-or-equal
    on odd frames   range = [1, 0.5],     compare = farther-or-equal
  ELSE
    clear depth ;  range = [0, 1] ;  compare = nearer-or-equal
  apply the range
```

**Invariants** —

- **The mirror scheme reserves half the depth range for the world and half for the reflection**, which is how a mirror
  is drawn without a stencil buffer: the reflected scene occupies depths the real scene cannot reach, so the two never
  interleave. This is the reason the depth range is a variable at all ([`glquake.h`](glquake.h.md)).
- **The alternating scheme skips the depth clear entirely** by flipping which end of the range is "near" and reversing
  the comparison every frame. Last frame's values are all *behind* everything this frame, so they never win a
  comparison. It halves the clear cost on hardware where clearing was expensive. It breaks if any frame is drawn
  without covering the whole viewport, which is why it is a switch and not the default — and a rebuild on modern
  hardware should not bother.
- The clear of the *colour* buffer is separately optional, because the world is assumed to cover the viewport. Leaving
  it off shows garbage at the edges of a map with a leak, which is how mappers found leaks.

## `R_SetupFrame`, `R_SetFrustum`, `SignbitsForPlane`, `R_CullBox`

**Contract** — advance the frame counter, derive the view basis from the angles, find the leaf the viewer is in,
animate the light styles, push the dynamic lights into the world, and reset the counters; build the four side planes of
the view volume from the field of view; precompute a plane's sign pattern; and reject a bounding box that lies wholly
outside any of the four planes.

**Invariants** — the frustum planes are the same four-plane test as the software renderer
([`r_main.c`](r_main.c.md)), with the sign-pattern trick for box rejection unchanged: **the plane's signs select which
corner of the box to test**, so one dot product decides instead of eight. There is no near or far plane in the test,
because the projection handles those.

## `MYgluPerspective`, `R_SetupGL`

**Contract** — build a perspective projection from a vertical field of view, an aspect ratio and the depth limits; and
set the viewport to the view rectangle, install the projection, then install the view transform: rotate the world so
the axis convention matches the game's, then rotate by the negated view angles, then translate by the negated view
origin.

**Invariants** —

- The perspective matrix is **written out rather than taken from the library's utility**, so the near and far planes are
  exactly what the engine chooses. A rebuild should note the values: the near plane is very close and the far plane far
  enough for the largest map, and the ratio between them is what determines depth precision — the thing the mirror
  scheme above is already spending half of.
- The view transform is built as **rotate-then-translate with everything negated**, which is the inverse of placing a
  camera. Composing it the other way is the most common rebuild error here.
- Two fixed rotations convert between the game's axis convention — forward along one horizontal axis, up along the
  vertical — and the library's. A rebuild picks one convention and applies the conversion once, here.
- The viewport is computed by **scaling the view rectangle from the engine's logical resolution to the window's**,
  because the two need not match: the engine's 2D layer draws at a chosen logical size and the window may be larger.

## `R_RenderScene`

**Contract** — draw the world, then every entity on the visible list, then the sky.

## `R_DrawEntitiesOnList`

**Contract** — walks the visible entity list, dispatching each to the model, brush-model or sprite path by its model's
kind. Draws models in two passes when shadows are on: all models, then all shadows.

**Invariants** — shadows are a **second pass over the same list**, because each shadow is a blended flat polygon and
they must all land after every solid model or a model would be drawn over its neighbour's shadow.

## `R_RotateForEntity`

**Contract** — appends the entity's translation and its three rotations to the current transform.

**Invariants** — the **rotation order is fixed and one axis is negated**. That negation is the engine's angle
convention ([`mathlib.c`](mathlib.c.md)) and must match everywhere or entities face the wrong way.

## `GL_DrawAliasFrame`

**Contract** — submits one pose of an animated model. Walks a precompiled command list of triangle strips and fans,
taking texture coordinates from the command list and positions and normals from the pose, computing one intensity per
vertex from its normal.

```text
FUNCTION draw_alias_frame(model, pose)
  verts = model.poses[pose]
  order = model.commands            # built once at load, see gl_mesh.c
  LOOP
    count = next command
    IF count == 0  DONE
    IF count < 0   begin a fan of -count vertices
    ELSE           begin a strip of count vertices
    REPEAT count times
      texture coordinate = next two values from the command list
      intensity = shadedots[ vertex.normal_index ] * shadelight
      colour = (intensity, intensity, intensity)
      position = vertex.position          # NOTE: the model's own integer scale
      emit
    end the primitive
```

**Invariants** —

- **Texture coordinates live in the command list, positions in the pose.** One command list serves every pose of the
  model, because the topology and the texture mapping never change between poses — only the positions do. That is the
  whole point of [`gl_mesh.c`](gl_mesh.c.md), and it is what makes an animated model cheap.
- A negative count means a fan and a positive count a strip. A single signed integer carries both the primitive kind
  and the length.
- **Lighting is one intensity per vertex, looked up from a table by the vertex's normal index.** The model format
  ([`modelgen.h`](modelgen.h.md)) stores a normal as an index into a fixed set of 162 directions, and the engine
  precomputes, for each of sixteen light directions, the dot product of every one of those 162 normals. So per-vertex
  lighting costs one table read. This is the single best trick in the model renderer and it survives any rebuild: a
  quantized normal set plus a per-light-direction table is still cheaper than a dot product, and it is why the format
  quantizes normals in the first place.
- The light direction is quantized to those sixteen, so a model's shading **snaps** as it turns. That is visible and
  characteristic.
- The positions are submitted **unscaled**, with the model's scale and offset folded into the transform by the caller.
  So the vertices go out as the small integers the file stores.

## `GL_DrawAliasShadow`

**Contract** — submits the same command list as a flat polygon on the floor: each vertex is scaled into model space,
displaced horizontally in proportion to its height above the light's floor point, and flattened to a fixed height just
above the floor.

```text
FUNCTION draw_alias_shadow(model, pose)
  lheight = entity.origin.z - the floor point found by the light trace
  height  = 1 - lheight                      # just above the floor
  FOR EACH vertex of the command list
    point = vertex scaled and offset into model space
    point.xy -= shadevector.xy * (point.z + lheight)
    point.z  = height
    emit
```

**Invariants** — the shadow is a **projection of the model onto a single horizontal plane**, at the height of the floor
found by the lighting trace ([`gl_rlight.c`](gl_rlight.c.md)). It is not a real shadow: it assumes a flat floor, it
self-overlaps, and it is drawn blended with depth writing off so the overlaps do not darken. Standing on stairs makes it
visibly wrong. Recorded as a decision, not a defect: it is a shadow for one tenth the cost, and it looks right in the
common case.

The fixed height above the floor is the classic co-planar bias, and the value matters: too small and the shadow
z-fights with the floor, too large and it floats.

## `R_SetupAliasFrame`, `R_DrawAliasModel`

**Contract** — pick the pose from the frame number and, for a grouped frame, from the time within its interval; and draw
one animated model: reject it by bounding sphere, compute its lighting from the static light at its origin plus every
dynamic light in range, halve the light on a player's own body seen from a mirror, select the light-direction table
from the entity's yaw, bind the skin, install the entity transform with the model's scale folded in, and submit the
pose — interpolating nothing.

**Invariants** —

- **Lighting is per entity, not per vertex position**: one intensity and one direction for the whole model, sampled at
  its origin. A model straddling a light boundary is lit uniformly. This is why the tables above are indexed by normal
  alone.
- The light direction table is chosen from the entity's **yaw only**, and the light is then treated as coming from a
  fixed elevation. So models are never lit from above or below.
- A dynamic light adds to the *ambient* term and moves the direction toward itself, which is why a rocket lights a
  monster evenly rather than from the side.
- A player model has its **skin replaced by a per-player translated copy** ([`gl_rmisc.c`](gl_rmisc.c.md)), addressed
  by the reserved handle range.
- There is **no interpolation between poses**. Models animate at their stored frame rate, snapping. Every later engine
  added interpolation here, and a rebuild may; the recipe records that the original does not, because the snapping is
  part of how the game looks.

## `R_GetSpriteFrame`, `R_DrawSpriteModel`

**Contract** — select a sprite frame by time within its group, then draw it as a single quad oriented by the sprite's
declared type: facing the viewer, facing the viewer but upright, oriented by its own angles, or oriented about a fixed
axis.

**Invariants** — the four orientation types come from the sprite format ([`spritegn.h`](spritegn.h.md)) and each builds
its quad from a different pair of basis vectors. The **upright-facing** type is what keeps a torch flame vertical while
still turning to face the viewer, and it is the one that needs the view basis projected onto the horizontal plane
rather than used directly.

Sprites are drawn with alpha testing rather than blending, so they need no sorting — a decision that costs soft edges
and buys order independence.

## `R_DrawViewModel`

**Contract** — draws the weapon, unless the player is dead, invisible, in the chase camera, or an environment map is
being captured. Lights it from the point light at the player's origin with a floor under the value, adds nearby dynamic
lights, and draws it into a **compressed depth range**.

**Invariants** — **the depth range is squeezed to the nearest third for this one model**, so the weapon is always in
front of the world no matter how close a wall is. That is the whole trick, and it is the cheapest solution to the
oldest problem in first-person rendering. A rebuild can do the same or render the weapon in a second pass with its own
projection; the recipe records that a depth-range squeeze is sufficient.

The light floor — never darker than a fixed minimum — exists because a weapon invisible in a dark room is a usability
failure, not a lighting success.

## `R_PolyBlend`

**Contract** — draws a single large quad in front of the viewer in the current tint colour, with depth testing and
texturing off, when the tint is not transparent.

**Invariants** — the tint colour and its strength come from [`view.c`](view.c.md) and encode damage, powerups and
liquid immersion. Drawing it as a **quad in world space in front of the camera** rather than a screen-space rectangle
means the 2D layer need not be entered, which is the only reason to do it this way; a rebuild may just draw a screen
rectangle.

## `R_Mirror`

**Contract** — when the map contains a mirror surface: reflect the view origin and direction through the mirror's
plane, re-derive the view angles from the reflected direction, add the player's own body to the visible list, render
the whole scene again into the *far* half of the depth range, then blend the mirror surface itself over the result at
the player's chosen opacity, with the projection mirrored and the face winding reversed.

```text
FUNCTION mirror()
  save the world transform
  d = distance from the view origin to the mirror plane
  view origin    -= 2d * plane normal            # reflect the point
  view direction -= 2 * (direction . normal) * normal
  re-derive pitch and yaw from the reflected direction ;  negate roll
  add the player's own entity to the visible list       # you see yourself
  depth range = the far half ;  render the scene and the liquids
  depth range = the near half
  mirror one axis of the projection and reverse the face winding
  restore the saved transform
  draw the mirror's own surfaces blended at the chosen opacity
```

**Invariants** —

- **The projection is mirrored, so the winding of every triangle flips and the culling direction must be reversed.**
  Which axis to mirror depends on whether the mirror plane is horizontal. Getting this wrong makes the reflection
  render inside-out — every back face visible and every front face gone — and it is the classic mirror bug.
- The angles are **re-derived from the reflected direction vector** rather than reflected as angles, because reflecting
  Euler angles through an arbitrary plane has no simple form. Roll is negated separately.
- The player's body is added to the visible list only for this pass, which is how you see yourself in a mirror in a
  game that otherwise never draws your own body.
- Only one mirror surface per map is supported, identified by its texture name at load. A rebuild wanting several needs
  a pass per mirror and a depth-range slice per pass — which is where this scheme runs out and a stencil buffer starts
  paying.

**Notes** — the mirror is not used by any shipped map. It is worth keeping in the recipe because it is a complete,
working, portal-free reflection technique for hardware with no stencil, and because it explains the depth-range
partitioning that the rest of the file is built around.
