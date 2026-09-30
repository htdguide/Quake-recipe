# WinQuake/r_alias.c

> Draws an animated model: builds its transform, culls it, dequantizes and projects every vertex into fixed point with a table-looked-up brightness, and chooses among three levels of rasterization detail.

**Needs** — [`r_local.h`](r_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [`modelgen.h`](modelgen.h.md) · [`anorms.h`](anorms.h.md) · [`anorm_dots.h`](anorm_dots.h.md) · [`client.h`](client.h.md) · [`mathlib.h`](mathlib.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops) ([`r_aliasa.s`](r_aliasa.s.md))
**Used by** — [`r_main.c`](r_main.c.md) calls the cull and the draw; [`d_polyse.c`](d_polyse.c.md) rasterizes
**Tier floor** — none

## Purpose

The model pipeline: cull, transform, project, light, clip, rasterize. Two things are worth attention.

**The transform folds the projection into itself** so that a vertex becomes screen coordinates and a depth
reciprocal in one matrix multiply and one divide, with the depth scaled so that its reciprocal lands in
31 bits for free.

**There are three levels of detail**, chosen by the model's on-screen size: full triangle rasterization,
recursive subdivision, and plotting individual vertices as points. The thresholds are in
[`r_local.h`](r_local.h.md) and the third level is why distant monsters visibly disintegrate.

## State

```text
CONSTANT light_min = 5                       # the darkest a model may be
CONSTANT numvertexnormals = 162
VARIABLE aedges : AEdge[12]                  # the bounding box's twelve edges
VARIABLE aliastransform : real[3][4]         # the folded transform
VARIABLE pfinalverts : FinalVert             # the output vertex array
VARIABLE pauxverts : AuxVert                 # their view-space positions
VARIABLE r_amodels_drawn : int
VARIABLE a_skinwidth, numverts, numtriangles : int
VARIABLE paliashdr : AliasHeader ;  pmdl : ModelHeader
VARIABLE leftclip, topclip, rightclip, bottomclip : real
VARIABLE r_acliptype : int
```

## `R_AliasCheckBBox`

**Contract** — takes nothing; culls the current entity's model against the frustum using its bounding box's
twelve edges, and records whether it was **trivially accepted** — entirely inside, so that clipping can be
skipped. Returns whether anything is visible.

**Invariants** — the trivial-accept result is stored on the entity
([`render.h`](render.h.md)) and read by the drawing path to select the unclipped vertex preparation. That is the
single largest saving in the model pipeline: a fully-visible model skips per-triangle clipping entirely.

The box comes from the model record, which for every animated model is a hard-coded ±16 units
([`model.c`](model.c.md#mod_loadaliasmodel)) — so the cull is wrong for any model larger than that, and large
monsters are culled while still partly visible.

## `R_AliasSetUpTransform`

**Contract** — builds the matrix that takes a quantized model vertex to view space with the projection folded
in, from the entity's angles, its origin, the model's scale and origin, and the projection scales. Takes the
trivial-accept status, which selects a variant with the screen scaling included.

```text
FUNCTION r_alias_set_up_transform(trivial_accept)
  # A rotation from the entity's three angles, then the model's own scale and
  # origin, then the translation to the view-relative position.
  build the rotation from currententity.angles
  fold in the model's scale and scale origin
  fold in the translation (the entity's origin minus the view position)
  IF trivial_accept
    # Also fold in the screen scaling, AND scale down z so that its reciprocal
    # lands in 31 bits with no further shifting, and scale x and y so that
    # the 16.16 texture coordinates come out directly.
    fold in aliasxscale, aliasyscale and the depth scaling
```

**Invariants** — the comment states the trick precisely: **z is scaled down so that one over z is scaled 31
bits for free**, and x and y are scaled so the projected coordinates land in the fixed-point format the
rasterizer wants. So the per-vertex work is a matrix multiply, one divide, and two multiplies — no shifting and
no second scaling.

The transform is rebuilt **per model per frame**, and the source lists the same four improvements
[`r_bsp.c`](r_bsp.c.md#r_entityrotate-r_rotatebmodel) does: a table, storage with the entity, lazy caching, and sharing the
work. None was done.

## `R_AliasTransformAndProjectFinalVerts`

**Contract** — for every vertex of the current frame: dequantizes it, transforms it by the folded matrix,
divides by depth, produces screen coordinates, texture coordinates and a depth reciprocal as six fixed-point
integers, looks up the brightness from the vertex's normal index, and sets the clip flags.

```text
FUNCTION r_alias_transform_and_project_final_verts(fv, stverts)
  FOR EACH vertex i
    # Transform. The matrix already holds the scale and the projection.
    z = the transformed z
    zi = 1 / z
    fv[i].v[5] = truncate(zi * 0x8000 * 0x10000)      # the depth reciprocal
    fv[i].v[0] = truncate(x * zi) + aliasxcenter      # screen u
    fv[i].v[1] = truncate(y * zi) + aliasycenter      # screen v
    fv[i].v[2] = stverts[i].s                        # already 16.16 at load
    fv[i].v[3] = stverts[i].t
    fv[i].v[4] = shadedots[the light yaw][vertex's normal index] * shadelight
                 + ambientlight                       # ONE table lookup
    fv[i].flags = which screen edges it is outside, plus the seam flag
```

**Invariants** — three things.

**Lighting is one indexed load and one multiply-add** ([`anorm_dots.h`](anorm_dots.h.md)): the table gives a
multiplier for the vertex's normal against the quantized light direction, and the ambient floor is added. So
the whole of model lighting is that.

**The texture coordinates were converted to 16.16 at load**
([`model.c`](model.c.md#mod_loadaliasmodel)), so they pass through unchanged.

**The clip flags and the seam flag share one field**
([`r_shared.h`](r_shared.h.md)), which is why the seam flag's value must not collide with a clip bit.

## `R_AliasPreparePoints`, `R_AliasPrepareUnclippedPoints`

**Contract** — the clipped and unclipped vertex paths. The unclipped one transforms and projects every vertex
and hands the whole mesh to the rasterizer. The clipped one transforms every vertex into view space, then walks
the triangles clipping each against the frustum and emitting the survivors.

**Invariants** — the split is the payoff of the trivial-accept test. The unclipped path is a single pass over
the vertices; the clipped one is a per-triangle clip with vertex generation.

## `R_AliasClipTriangle`

**Contract** — in [`r_aclip.c`](r_aclip.c.md): clips one triangle against the frustum, producing up to several
triangles.

## `R_AliasSetupSkin`

**Contract** — selects the model's skin for the entity's skin number, resolving a skin group by its intervals
and the clock, and builds the per-row address table
([`d_polyse.c`](d_polyse.c.md#d_polysetupdatetables)).

## `R_AliasSetupFrame`

**Contract** — selects the model's frame for the entity's frame number, resolving a frame group by its
cumulative intervals and the clock, offset by the entity's animation phase.

```text
FUNCTION r_alias_setup_frame()
  frame = currententity.frame, CLAMPED into range with a diagnostic
  IF the frame is a single  use it
  ELSE
    # A group: find the first interval whose cumulative time exceeds the
    # entity's own phase-offset clock.
    t = (cl.time + currententity.syncbase) MODULO the group's total interval
    pick the first frame whose interval exceeds t
```

**Invariants** — **the phase offset comes from the entity**, set at spawn either to zero or to a random value
by the model's synchronization type ([`modelgen.h`](modelgen.h.md)). That is what makes a row of torches
flicker out of step.

## `R_AliasSetupLighting`

**Contract** — takes the ambient and directional strengths and the light direction from the caller; quantizes
the direction's yaw into one of sixteen table rows, and enforces the minimum brightness.

**Invariants** — the light direction is quantized to sixteen yaw steps and its elevation is discarded
([`anorm_dots.h`](anorm_dots.h.md)), which is why a model's shading steps as it turns.

The minimum of 5 exists so a model in a black room is still faintly visible — the comment says so.

## `R_AliasDrawModel`

**Contract** — draws the current entity's model: sets up the lighting, the frame, the skin and the transform;
chooses among the three detail levels by the model's projected size against the transition distance; prepares
the points; and calls the rasterizer.

```text
FUNCTION r_alias_draw_model(lighting)
  r_alias_setup_lighting(lighting) ;  r_alias_setup_frame()
  r_alias_setup_skin()
  # Choose the detail level from the model's own average triangle size scaled
  # by its distance, against the resolution-scaled transition distance.
  IF the model is small enough on screen
    r_affinetridesc.drawtype = SUBDIVIDING            # or point plotting
  ELSE
    r_affinetridesc.drawtype = FULL RASTERIZATION
  r_alias_set_up_transform(currententity.trivial_accept)
  IF trivially accepted  r_alias_prepare_unclipped_points()
  ELSE                   r_alias_prepare_points()
  d_polyset_draw()
```

**Invariants** — **the detail decision uses the model's own stored average triangle size**
([`modelgen.h`](modelgen.h.md)) scaled by the transition constant
([`r_local.h`](r_local.h.md)), whose derivation is "about one pixel per triangle". So the switch happens when
triangles reach pixel size, which is exactly when the cheaper path becomes indistinguishable — in principle.
In practice the transition is visible, and a rebuild may simply always rasterize.

**Notes** — the source's notes here are the densest in the tree: it wants the vertex loop in assembly, the
transform stored with the entity, the concatenation done more cheaply, and several globals hoisted. All are
performance, none behaviour.
