# WinQuake/r_main.c

> The renderer's frame: derives the projection from the view, marks visible leaves from the precomputed visibility, then runs the five passes — world, sub-models, entities, weapon, particles — in an order that is itself the architecture.

**Needs** — [`r_local.h`](r_local.h.md) · [`r_shared.h`](r_shared.h.md) · [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [`client.h`](client.h.md) · [`render.h`](render.h.md) · [`view.h`](view.h.md) · [`screen.h`](screen.h.md) · [`sound.h`](sound.h.md) · [`zone.h`](zone.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`view.c`](view.c.md) calls the frame; [`host.c`](host.c.md) initializes it; [`model.c`](model.c.md) uses its placeholder texture
**Tier floor** — T1 as written; the pools are stack allocations and the projection's precomputation is a T1 artifact

## Purpose

Two things live here. The **projection derivation** — how a field of view and a rectangle become the twenty
precomputed values [`render.h`](render.h.md) declares, plus four clip planes and a model-detail
transition distance. And the **frame's pass order**, which is where the whole architecture becomes visible:
the world is drawn first with no depth reads, then the depth test is turned on, then everything else.

## State

```text
VARIABLE r_refdef : RefDef                    # the view description
VARIABLE xcenter, ycenter, xscale, yscale, xscaleinv, yscaleinv : real
VARIABLE xscaleshrink, yscaleshrink : real
VARIABLE aliasxscale, aliasyscale, aliasxcenter, aliasycenter : real
VARIABLE screenedge : MPlane[4]               # the four view-frustum planes
VARIABLE pixelAspect, screenAspect, verticalFieldOfView : real
VARIABLE r_framecount : int = 1               # starts at 1, so that a
                                              # zero-initialized stamp never
                                              # matches
VARIABLE r_visframecount : int
VARIABLE r_viewleaf, r_oldviewleaf : MLeaf
VARIABLE r_cnumsurfs, r_numallocatededges : int
VARIABLE r_surfsonstack : bool
VARIABLE auxedges : Edge                      # a heap pool, when the stack one
                                              # is too small
VARIABLE r_stack_start : pointer               # for the stack-depth check
VARIABLE r_aliastransition, r_resfudge : real  # the model detail thresholds
VARIABLE r_fov_greater_than_90 : bool
VARIABLE d_lightstylevalue : int[256]          # each style's level, 8.8 fixed
VARIABLE twenty tunable variables               # see r_local.h
```

**Invariants** — **the frame counter starts at 1** so that a structure zeroed at allocation never appears to
have been visited this frame. Every frame-stamp comparison in the renderer depends on it.

## `R_InitTextures`

**Contract** — builds the placeholder texture: a 16-by-16 black-and-white checkerboard with all four mip
levels, substituted for any texture a map references but does not contain
([`model.c`](model.c.md#mod_loadtexinfo)).

**Invariants** — called **even on a dedicated server** ([`host.c`](host.c.md#host_init)), because the map
loader needs it whether or not anything will draw.

## `R_Init`

**Contract** — records the stack position for the later depth check, builds the warp sine tables, registers
two commands and twenty variables.

## `R_InitTurb`

**Contract** — builds the two sine tables the liquid and screen warps index.

```text
FUNCTION r_init_turb()
  FOR EACH i IN 0 .. table size - 1
    sintable[i]    = amp  + sin(i * 2*pi / 128) * amp      # amp  = 8 * 65536
    intsintable[i] = amp2 + sin(i * 2*pi / 128) * amp2     # amp2 = 3
```

**Invariants** — both tables are **offset by their amplitude** so every entry is non-negative, which lets the
warps add them to a coordinate without a sign concern. The cycle is 128 entries, and the tables are longer
than one cycle by the maximum display dimension so that a warp can index from any phase without wrapping
([`r_shared.h`](r_shared.h.md)).

The two amplitudes serve different purposes: the large one is the liquid surface's texture displacement in
16.16 fixed point, the small one is the screen warp's pixel displacement. The source's comment notes the
second uses the amplitude and not the speed — a correction to an earlier slip.

## `R_NewMap`

**Contract** — prepares for a level: clears every leaf's entity-fragment list and the particle pool, then
sizes the surface and edge pools — using the stack when the configured size fits and the hunk otherwise.

```text
FUNCTION r_new_map()
  FOR EACH leaf IN the world  clear its entity-fragment list
  r_viewleaf = nothing ;  clear the particles

  r_cnumsurfs = max(the configured surface count, 800)
  IF r_cnumsurfs > 800
    surfaces = hunk-allocate that many
    surfaces = surfaces - 1                   # so that index 1 is the first
                                              # real entry; index 0 must be
                                              # unusable
    r_surfsonstack = false ;  patch the assembly's base pointer
  ELSE
    r_surfsonstack = true                     # allocated per frame, on the stack

  r_numallocatededges = max(the configured edge count, 2400)
  auxedges = nothing IF that fits on the stack ELSE a hunk allocation
```

**Invariants** — **the surface pool pointer is decremented by one entry** so that the array is effectively
one-based. Index 0 must not name a real surface, because an edge stores zero to mean "no surface"
([`r_shared.h`](r_shared.h.md)). A rebuild that reserves a real entry 0 instead loses nothing.

The stack-versus-heap choice is a memory economy: 800 surfaces at 64 bytes and 2400 edges at 32 is about 128
kilobytes that costs nothing when it lives on the stack. A player who raises the limits pays hunk memory
instead.

## `R_SetVrect`

**Contract** — computes the view rectangle from the display's rectangle, a number of lines to reserve at the
bottom, and the view-size setting. Forces a full-screen view at the end-of-level screen. Enforces a minimum
width of 96 and rounds the width to a multiple of eight and the height to a multiple of two, centring the
result.

```text
FUNCTION r_set_vrect(display, lineadj) -> view
  size = min(the view-size setting, 100)
  IF at the intermission  size = 100 ;  lineadj = 0
  size = size / 100
  h = display.height - lineadj
  view.width = display.width * size
  IF view.width < 96  view.width = 96 ;  size = 96 / display.width
  view.width = view.width ROUNDED DOWN to a multiple of 8
  view.height = min(display.height * size, display.height - lineadj)
  view.height = view.height ROUNDED DOWN to a multiple of 2
  view.x = (display.width - view.width) / 2
  view.y = (h - view.height) / 2
  IF the stereo mode is on  halve view.y and view.height
```

**Invariants** — the **width is a multiple of eight** so that span fillers can assume it, and the **height is
even** so that the stereo mode can halve it. The minimum of 96 is described as the minimum for icons — below
it the status bar's graphics would not fit.

The line reservation is the status bar's height ([`sbar.h`](sbar.h.md)), so shrinking the view and showing
the bar are one mechanism.

## `R_ViewChanged`

**Contract** — recomputes every derived value from the view description and the display's pixel aspect: the
rectangle, its twenty precomputed edge forms, the projection scales and centre, the four frustum planes, the
model detail transition distances, and the wide-field-of-view flag. Then patches the assembly and notifies
the rasterizer. Must be called before the first frame after any change.

```text
FUNCTION r_view_changed(display_rect, lineadj, aspect)
  r_refdef.vrect = r_set_vrect(display_rect, lineadj)

  # The field of view, expressed as visible width at unit depth.
  r_refdef.horizontalFieldOfView = 2 * tan(fov_x / 2 IN radians)

  # Twenty precomputed forms of the rectangle's edges: float copies, copies
  # biased by half a pixel for clamping, and two pre-shifted into 12.20 fixed
  # point with a half-pixel bias minus one.
  ...

  # The model-detail rectangle, scaled by the rasterizer's requested factor.
  r_refdef.aliasvrect = r_refdef.vrect SCALED BY r_aliasuvscale

  pixelAspect = aspect
  screenAspect = view.width * pixelAspect / view.height
  verticalFieldOfView = horizontalFieldOfView / screenAspect

  # The projection centre, biased by half a pixel. See the note.
  xcenter = view.width  * 0.5 + view.x - 0.5
  ycenter = view.height * 0.5 + view.y - 0.5
  xscale = view.width / horizontalFieldOfView ;  xscaleinv = 1/xscale
  yscale = xscale * pixelAspect              ;  yscaleinv = 1/yscale
  xscaleshrink = (view.width - 6) / horizontalFieldOfView
  yscaleshrink = xscaleshrink * pixelAspect

  # The four frustum planes, in VIEW space: each has z = 1 and an x or y
  # component that is the reciprocal of the half-field-of-view.
  screenedge[0] = { normal: (-1/(xOrigin*hfov),        0, 1) }    # left
  screenedge[1] = { normal: ( 1/((1-xOrigin)*hfov),    0, 1) }    # right
  screenedge[2] = { normal: (0, -1/(yOrigin*vfov),        1) }    # top
  screenedge[3] = { normal: (0,  1/((1-yOrigin)*vfov),    1) }    # bottom
  normalize all four

  # The model detail thresholds, scaled by resolution and field of view.
  res_scale = sqrt(view area / (320*152)) * (2 / horizontalFieldOfView)
  r_aliastransition = the configured base * res_scale
  r_resfudge        = the configured adjustment * res_scale

  r_fov_greater_than_90 = (the field of view setting > 90)
  patch the surface-builder assembly and select the shading table
  d_view_changed()
```

**Invariants** — six load-bearing decisions.

**The projection centre is biased by minus half a pixel**, and the comment explains at length: if the
arithmetic were exact the values would span 0.5 to range-plus-0.5, and the bias moves them into
0.000001 to range-plus-0.999999 so that truncation renders the last row and column but never the first.
That makes the fill exactly edge to edge. Getting the bias wrong loses a row or draws one outside.

**The two pre-shifted edge values are biased by half a pixel minus one**, for the same reason in fixed
point.

**The frustum planes are expressed with z = 1 and a reciprocal-field-of-view component**, then normalized —
which is the cheapest construction of a view-space frustum plane and is why the clipper can dot against them
directly.

**The planes are placed using the projection origin, not the centre**, so the vertical asymmetry
([`render.h`](render.h.md)) is respected: the top and bottom planes are at different angles.

**The model detail transition scales with the square root of the view's area and inversely with the field of
view.** So a larger or narrower view keeps models detailed further out, which keeps their apparent detail
constant. The reference area is 320 by 152 — the base resolution minus the status bar.

**The shrink scales exist for the weapon model**, drawn six pixels narrower so that its edges cannot poke
outside the view.

The two self-modifying patches ([`sys.h`](sys.h.md#sys_makecodewriteable)) select the eight- or sixteen-bit
surface builder and the matching shading table.

## `R_MarkLeaves`

**Contract** — when the view has moved to a different leaf, marks every leaf in the new leaf's visibility set,
and every node on the path from each such leaf up to the root, with the current visibility stamp.

```text
FUNCTION r_mark_leaves()
  IF the view leaf has not changed  RETURN         # nothing to do
  r_visframecount = r_visframecount + 1
  r_oldviewleaf = r_viewleaf
  vis = the view leaf's visibility set, decompressed
  FOR EACH leaf index i
    IF bit i OF vis IS SET
      node = leaf i+1                              # the set excludes leaf 0
      WHILE node EXISTS AND node.visframe != r_visframecount
        node.visframe = r_visframecount
        node = node.parent                         # walk UP to the root
```

**Invariants** — **this is the only use of the precomputed visibility**, and it is what makes the tree walk
cheap: the walk ([`r_bsp.c`](r_bsp.c.md)) descends only into nodes carrying the current stamp, so it visits
only the part of the tree leading to potentially-visible leaves.

**The upward walk stops at the first already-marked node**, which is what keeps the marking linear rather
than quadratic: once a subtree's root is marked, every other leaf in it terminates immediately.

**The parent pointers this needs are added at load** ([`model.c`](model.c.md#mod_loadnodes-mod_setparent)) and exist for
exactly this purpose.

**It runs only when the view leaf changes**, so standing still costs nothing. That is also why it must be
called before anything that needs to know whether the view is in water.

## `R_DrawEntitiesOnList`

**Contract** — draws every entity in the client's visible list except the player's own, dispatching by model
kind. For an animated model, culls it against the frustum, computes its lighting from the world's baked
light plus every nearby dynamic light, clamps the result, and draws.

```text
FUNCTION r_draw_entities_on_list()
  IF entity drawing is disabled  RETURN
  FOR EACH visible entity
    IF it IS the player's own entity  CONTINUE     # never draw yourself
    SELECT its model's kind
      sprite:
        r_entorigin = its origin ;  modelorg = r_origin - r_entorigin
        r_draw_sprite()
      alias:
        r_entorigin = its origin ;  modelorg = r_origin - r_entorigin
        IF NOT r_alias_check_bbox()  CONTINUE      # culled; also sets the
                                                   # trivial-accept status
        j = the world's baked light AT its origin
        lighting.ambientlight = j ;  lighting.shadelight = j
        lighting.plightvec = (-1, 0, 0)            # a FIXED light direction
        FOR EACH live dynamic light
          add = its radius - the distance TO the entity
          IF add > 0  lighting.ambientlight = lighting.ambientlight + add
        IF ambientlight > 128  ambientlight = 128
        IF ambientlight + shadelight > 192  shadelight = 192 - ambientlight
        r_alias_draw_model(lighting)
```

**Invariants** — five things, and two are confessions in the source.

**The light direction is a fixed constant**, pointing along negative x, and the source marks it for removal
in favour of real lighting. So every model in the game is lit from the same world direction regardless of
where the lights are. That is why models look flat and why a monster's shading does not change as it walks
past a lamp.

**The ambient and directional strengths are set to the same value**, sampled from the world's baked lighting
at the entity's origin ([`r_light.c`](r_light.c.md#recursivelightpoint-r_lightpoint)). So the model's *overall* brightness does
track the room, even though its *direction* does not.

**A dynamic light contributes only to the ambient term**, by radius minus distance with no falloff curve and
no direction. So a rocket lights a monster uniformly brighter rather than from one side.

**The clamps are 128 for ambient and 192 for the sum**, which is where the shading table's usable range ends
([`vid.h`](vid.h.md)). Without them a brightly lit model's shading indices leave the table.

**The player's own entity is skipped**, which is why you cannot see your own body in first person.

## `R_DrawViewModel`

**Contract** — draws the weapon in the player's hands, with the same lighting computation plus a floor of 24
so the weapon is never fully dark. Skipped when disabled, when the field of view exceeds 90 degrees, while
invisible, or when dead.

**Invariants** — **skipped above 90 degrees of field of view** because the weapon model is authored for that
framing and looks wrong wider. That is a content constraint enforced in code.

**A light floor of 24**, with the comment saying the gun should always have some light — because a weapon
invisible in a dark room is unusable.

The light *direction* here is the inverted view-up vector, unlike the world's fixed direction, so the weapon
is lit from the player's own perspective.

## `R_BmodelCheckBBox`

**Contract** — takes a sub-model and its bounds; returns clip flags saying which frustum planes it may
straddle, or a fully-clipped marker. A rotated sub-model is tested against its bounding sphere; an unrotated
one against its box.

**Invariants** — **a rotated sub-model falls back to a sphere test** because its box is not axis-aligned
after rotation. That is why the rotating-geometry path is conservative, and it is the same limitation
[`pr_cmds.c`](pr_cmds.c.md#setminmaxsize) reflects on the collision side.

## `R_EdgeDrawing`

**Contract** — allocates the edge and surface pools on the stack if they were not heap-allocated, runs the
world tree walk, turns on the rasterizer's depth comparison, draws the sub-models, then scans the edges into
spans.

```text
FUNCTION r_edge_drawing()
  edges    = the heap pool IF one exists ELSE a cache-aligned stack array
  surfaces = a cache-aligned stack array IF they live on the stack
             (again decremented by one so index 0 is unusable)
  r_begin_edge_frame()
  r_render_world()                  # the tree walk: emits edges and surfaces
  IF the rasterizer wants culled polygons  r_scan_edges()
  d_turn_z_on()                     # see the note
  r_draw_b_entities_on_list()       # sub-models: doors, platforms
  top up the sound mixer
  IF the rasterizer wants spans  r_scan_edges()
```

**Invariants** — the comment on the depth-enable call is the clearest statement of the architecture in the
tree: *only the world can be drawn back to front with no depth reads or compares, just depth writes.* So the
world pass writes depth and never reads it, and every later pass reads it. That is the asymmetry
[`d_local.h`](d_local.h.md#the-depth-buffer) describes, stated by its author.

**Sub-models are drawn into the same edge list as the world**, before the scan, which is why a door
interpenetrating a wall sorts correctly — and why the surface comparison needs its coplanar tie-break
([`r_edge.c`](r_edge.c.md#r_leadingedge)).

## `R_RenderView_`

**Contract** — draws one frame. Allocates the warp buffer on the stack, sets up the frame, marks visible
leaves, lowers the floating-point precision, then runs the passes: world and sub-models, entities, the
weapon, particles. Applies the underwater warp, sets the screen tint from the view leaf's contents, and
prints whatever diagnostics are enabled. Restores the floating-point precision.

```text
FUNCTION r_render_view_inner()
  warpbuffer = a 320 x 200 stack array ;  r_warpbuffer = it
  r_setup_frame()
  r_mark_leaves()                    # BEFORE anything, so we know if we are
                                     # in water
  sys_low_fp_precision()             # see the note
  IF there is no world model  FAIL WITH "NULL worldmodel"
  top up the sound mixer
  r_edge_drawing()                   # the world and sub-models
  top up the sound mixer
  r_draw_entities_on_list()          # models and sprites
  r_draw_view_model()                # the weapon
  r_draw_particles()
  IF the underwater warp is active  d_warp_screen()
  v_set_contents_color(the view leaf's contents)     # the screen tint
  print whatever diagnostics are enabled, including how many surfaces and
        edges were dropped
  sys_high_fp_precision()
```

**Invariants** — five things.

**The pass order is the architecture.** The world writes depth without reading; sub-models join the world's
edge list; then models, the weapon and particles each read depth and write it. Reordering any of the last
three changes what occludes what.

**The floating-point precision is lowered for the whole frame** and the comment says why: it makes division
fast, it reduces timing precision so it is not done globally, and it also sets truncation mode — which the
screen-area calculations depend on matching. So the mode change is not merely a speed choice, it is part of
the rounding contract ([`sys.h`](sys.h.md#floating-point-control)).

**The sound mixer is topped up three times per frame** — before the world, after it, and inside the span
flush ([`r_edge.c`](r_edge.c.md#r_scanedges)) — because a slow frame would otherwise starve the audio ring.
A rebuild with a callback-driven mixer deletes all of them.

**The warp buffer is a 64-kilobyte stack array**, allocated per frame.

**The dropped-surface and dropped-edge counts are reported**, with the edge count divided by two thirds
because each dropped polygon costs roughly one and a half edges. That instrumentation is how the pool sizes
were tuned.

## `R_RenderView`

**Contract** — wraps the frame in four assertions: that the stack is deep enough, that the hunk mark is
four-byte aligned, that the stack is aligned, and that the globals are aligned. Any failure is fatal.

**Invariants** — the stack-depth check compares the current frame's address against the one recorded at
initialization and fails if they differ by more than ten thousand bytes. The renderer allocates over a
hundred kilobytes of stack arrays and a 1996 stack was small, so being called from an unexpectedly deep
place was a real crash. A rebuild with heap pools deletes all four checks.
