# WinQuake/gl_rsurf.c

> The hardware world renderer: every surface's lightmap packed into a page at load time by a skyline allocator, surfaces drawn in texture order, and lighting applied either as a second texture unit in one pass or as a blended second pass.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`gl_model.h`](gl_model.h.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer)
**Used by** — [`gl_rmain.c`](gl_rmain.c.md) · [`gl_model.c`](gl_model.c.md) (lightmap build at load) · [`gl_screen.c`](gl_screen.c.md)
**Tier floor** — none

## Purpose

The hardware answer to the problem [`r_surf.c`](r_surf.c.md) and [`d_surf.c`](d_surf.c.md) solve in software: a world
surface has a base texture and a baked lightmap, and the two must be combined per pixel. Software combines them into a
new texture and caches it. Hardware cannot afford that — uploading a texture per surface per frame is far worse than
drawing twice — so this file instead **packs every lightmap in the map into a handful of large pages once**, and then
either multiplies the two textures in one pass or draws the geometry twice and blends.

That difference cascades: it is why [`gl_model.h`](gl_model.h.md) gives a surface a page coordinate instead of cache
slots, why draw order is by texture, and why the surface cache and its thrash detection disappear entirely.

## State

```text
CONSTANT block_width = 128 ;  block_height = 128     # one lightmap page
CONSTANT max_lightmaps = 64                          # pages
VARIABLE lightmap_bytes : 1, 2 or 4                  # chosen from what the library offers

VARIABLE lightmaps   : byte[4 * pages * 128 * 128]   # CPU copy of every page
VARIABLE allocated   : int[pages][128]               # skyline: used height per column
VARIABLE lightmap_modified : bool[pages]
VARIABLE lightmap_rectchange : rect[pages]           # dirty rectangle per page
VARIABLE lightmap_polys : list[pages]                # this frame's polygons per page
VARIABLE lightmap_textures                           # base handle of the page run
VARIABLE active_lightmaps : int
VARIABLE blocklights : unsigned[18*18]               # scratch for one surface
VARIABLE skychain, waterchain : surface list         # deferred kinds
```

**Invariants** —

- **A full copy of every page is kept in main memory.** The library offers no way to read a texture back, and a
  partial upload needs the surrounding bytes, so the CPU copy is the authority and the texture is a mirror of it. A
  rebuild on any API without texture readback needs the same shadow copy.
- A surface's lightmap is at most **18 by 18 samples**, because lightmap samples sit on a sixteen-unit grid
  ([`model.c`](model.c.md)) and a surface's extent is bounded by the map compiler. The scratch buffer is sized to that
  and nothing checks — an oversized surface corrupts memory. A rebuild should bound it.
- **A page is 128 by 128**, sixty-four of them. The size is a compromise: bigger pages waste less to packing but make
  each partial upload's row stride longer, and 128 was the largest guaranteed texture size on the hardware of the day.

## `AllocBlock`

**Contract** — takes a width and height in samples; finds room in the first page that has it, records the space as
used, and returns the page number and the position within it. Fatal error when every page is full.

```text
FUNCTION alloc_block(w, h) -> (page, x, y)
  FOR EACH page
    best = block_height
    FOR each candidate left edge i from 0 to block_width - w
      best2 = 0
      FOR each column j under the candidate
        IF allocated[page][i+j] >= best  REJECT this candidate
        best2 = MAX(best2, allocated[page][i+j])
      IF no column rejected it
        x = i ;  best = best2          # NOTE: best is lowered as candidates improve
    IF best + h > block_height  TRY the next page
    FOR each column under x  allocated[page][x+j] = best + h
    RETURN (page, x, best)
  FAIL WITH "full"
```

**Invariants** — this is a **skyline packer**: each page remembers, per column, how much height is consumed, and a
rectangle is placed at the lowest position where its whole width fits. It is the standard solution to packing many
small rectangles into few textures, and it is worth recognizing as such rather than reimplementing by feel.

The candidate loop **lowers its own ceiling as it goes** — once a placement at height *h* is found, later candidates
must beat it — so a single pass finds the best position rather than the first. Note the loop bound excludes a candidate
whose left edge is exactly `block_width - w`, so the rightmost legal column is never tried: a harmless off-by-one that
wastes a sliver of every page. A rebuild should use `<=` and need not reproduce the waste.

Packing happens **at map load**, in the order surfaces appear in the map, and is never undone. There is no free
operation, because a map's lightmaps live as long as the map.

## `GL_CreateSurfaceLightmap`, `GL_BuildLightmaps`

**Contract** — for one surface, allocate its rectangle, then build its lit samples into the page's CPU copy at that
position with the page's row stride; and for the whole world, clear the allocator, choose the sample format from what
the library supports, build every surface of the world and of every brush model, then upload all pages that received
anything.

**Invariants** — the **sample format is negotiated, not fixed**: four bytes, two bytes, or one byte per sample
depending on what the library exposes, and the build writes whichever was chosen. So the same builder produces
coloured or luminance-only lightmaps. A rebuild picks one format and deletes the negotiation.

Surfaces flagged as sky or as turbulent liquid get **no lightmap**, since neither is lit.

## `R_BuildLightMap`

**Contract** — takes a surface, a destination and a row stride; accumulates the surface's stored lightmap layers scaled
by their current style levels, adds every dynamic light touching it, then writes the samples out clamped and inverted
into the destination in the negotiated format. Records the style levels and the dynamic-light presence used, so the
caller can tell later whether they changed.

```text
FUNCTION build_lightmap(surf, dest, stride)
  samples = (extent / 16 + 1) in each axis
  IF the map has no light data OR full-bright is on
    fill the scratch with the maximum and skip to output
  ELSE
    clear the scratch
    FOR EACH of the surface's up-to-four light layers
      scale = the current level of that layer's style
      remember scale as the layer's cached level
      FOR EACH sample  scratch[sample] += stored sample * scale
    IF the surface is touched by dynamic lights this frame  add them
  FOR EACH sample
    value = scratch >> shift ;  clamp to the maximum
    write (maximum - value) into dest, in the negotiated format
  # a two- or four-byte format writes the same value in each channel
```

**Invariants** —

- The samples are written **inverted** — maximum minus value — because the combination in the hardware is a
  subtractive blend in the two-pass path, so a bright lightmap must be a *small* number. The one-pass multitexture
  path uses a modulate environment instead and therefore wants the *uninverted* value; the two paths agree because
  the blend factors are chosen to compensate. This is the subtlest thing in the file, and a rebuild that chooses one
  combination mode must make the inversion match it or every surface comes out as its own negative.
- **The style levels used are cached on the surface.** That, plus the dynamic-light flag, is the entire invalidation
  test: next frame, if a style moved or a dynamic light appeared or vanished, the rectangle is rebuilt and marked
  dirty. It is the same test as the software cache ([`r_surf.c`](r_surf.c.md)) with a different consequence.
- The four-layer limit and the sixteen-unit sample grid come from the map format
  ([`bspfile.h`](bspfile.h.md)) and are not negotiable.

## `R_AddDynamicLights`

**Contract** — for each dynamic light whose bit is set on the surface, project the light's origin onto the surface
plane, and add to every sample a value falling off with distance from the projected point, zero beyond the light's
radius.

```text
FUNCTION add_dynamic_lights(surf)
  FOR EACH light bit set on surf
    dist = distance from the light centre to the surface plane
    rad  = light radius - |dist|
    IF rad < light's minimum  SKIP                     # plane too far
    centre = light origin projected onto the plane
    local = centre expressed in the surface's texture axes, minus the surface origin
    FOR EACH sample (s, t)
      d = |local.t - t*16| + |local.s - s*16| / 2      # NOT a true distance
      ... (whichever axis distance is larger dominates)
      IF d < rad  scratch[sample] += (rad - d) * 256
```

**Invariants** — the falloff uses a **cheap approximation of distance**: the larger axis distance plus half the
smaller. It is not Euclidean and the resulting light pool is slightly diamond-shaped. This is a deliberate speed
choice, and it is *visible* — reproducing the look means reproducing the approximation, while a rebuild that wants
correctness can use a real distance and will get subtly rounder muzzle flashes.

The radius test is against the light's **plane distance** first, so a light on the far side of a wall costs one
subtraction.

## `R_RenderDynamicLightmaps`

**Contract** — called once per surface being drawn. Adds the surface's polygon to its page's list for this frame. If
any light style the surface uses has moved, or a dynamic light has appeared on it or just left it, rebuild its
rectangle in the CPU copy and grow the page's dirty rectangle to include it.

**Invariants** — **the dirty rectangle is a union over the whole page, tracked as top and height only**: the partial
upload always covers the full page width. Uploading a narrow tall strip was slower than a full-width one on the
hardware of the day, so width is not tracked. A rebuild on an API where a true sub-rectangle is cheap should track
both.

A surface that *had* a dynamic light last frame and has none now must also be rebuilt — the "just left" case — and
forgetting it leaves a muzzle flash burned into the wall. That is the rebuild bug this invariant exists to prevent.

## `R_BlendLightmaps`

**Contract** — the second lighting pass. For each page with polygons this frame: upload its dirty rectangle if any,
then draw every one of its polygons using the lightmap coordinates, with depth writing disabled and a subtractive
blend against what is already there. Restores the state afterwards.

**Invariants** — **depth writing is off but depth testing is on**, because the geometry being drawn is exactly the
geometry already in the depth buffer and a second write would be redundant at best.

The upload happens **inside the draw loop**, immediately before the page is used, which is what keeps the number of
uploads to at most one per page per frame no matter how many surfaces changed.

When the lighting-only debug switch is on the blend is replaced by a plain draw, which is how the lightmaps are
inspected.

## `R_DrawSequentialPoly`

**Contract** — the one-surface-at-a-time path, used when texture sorting is off. Draws a surface completely: base
texture, lightmap, and any distortion or sky treatment, choosing among a fixed set of cases — plain, plain with
multitexture, underwater-distorted, turbulent liquid, sky — and using one pass where the hardware has two texture
units and two where it does not.

**Invariants** — this exists **because a mirror and a transparent water surface cannot be drawn in texture order**:
they must be drawn at the right moment relative to what is behind them. So the engine keeps both a sorted path and a
sequential one and the player chooses. A rebuild with a modern pipeline keeps the sorted path for opaque geometry and
the sequential one for anything blended, which is the same conclusion reached from the other direction.

The multitexture variant is the interesting half: the base texture on one unit, the lightmap on the other, combined by
the hardware, one pass, no blending. That is the whole reason the extension was worth detecting
([`glquake.h`](glquake.h.md)).

## `DrawGLPoly`, `DrawGLWaterPoly`, `DrawGLWaterPolyLightmap`

**Contract** — submit one polygon as a fan; or submit it with each vertex's position displaced by a time-varying
function of its other two coordinates, for the underwater view; or the same displacement with lightmap coordinates.

**Invariants** — the underwater displacement is applied **per vertex of the already-subdivided polygon**
([`gl_warp.c`](gl_warp.c.md) does the subdivision), which is why subdivision exists: without it a wall would bend at
its corners only.

## `R_RenderBrushPoly`

**Contract** — draws one world or brush-model surface: picks the animated texture frame, binds it, defers sky and
liquid surfaces to their chains or draws them directly, submits the polygon, and hands the surface to the dynamic
lightmap check.

## `R_TextureAnimation`

**Contract** — takes a texture; returns the frame of its animation group that is current, following the alternate
group when the entity requests it.

**Invariants** — the animation advances on **world time divided by a tenth of a second**, the same interval as the
game's animation opcode ([`pr_exec.c`](pr_exec.c.md)), and the frame is chosen by a modulus over the group so a group
of any length works. The alternate group is selected by an entity field, which is how a switchable texture works
without the renderer knowing about switches.

## `DrawTextureChains`, `R_DrawWorld`, `R_RecursiveWorldNode`, `R_MarkLeaves`

**Contract** — walk the world tree front to back marking visible surfaces onto their texture chains, then draw the
chains texture by texture, then the lighting pass, then sky and water; and mark the leaves the viewer can see from the
precomputed visibility set.

```text
FUNCTION recursive_world_node(node)
  IF the node is outside the visible set OR outside the frustum  RETURN
  IF the node is a leaf  mark its surfaces as visible ;  RETURN
  side = which side of the node's plane the viewer is on
  recursive_world_node(child on the viewer's side)
  FOR EACH surface on the node
    IF it faces away from the viewer  SKIP
    IF it is outside the frustum  SKIP
    add it to its texture's chain
  recursive_world_node(the other child)
```

**Invariants** — the traversal is **front to back**, exactly as in [`r_bsp.c`](r_bsp.c.md), even though a depth buffer
makes the order unnecessary for correctness. It is kept because front-to-back maximizes early depth rejection: a
surface hidden behind one already drawn is discarded by the hardware without being textured. A rebuild should keep it
for the same reason.

But note the **conflict**: surfaces are collected into texture chains and then drawn per texture, which destroys the
front-to-back order the traversal established. The engine accepts this — texture changes cost more than overdraw —
which is exactly the trade-off a rebuild must re-evaluate on its own hardware, where the answer is often the opposite.

Visibility marking, frustum rejection and the back-face test are unchanged from the software renderer; read
[`r_bsp.c`](r_bsp.c.md) for them.

## `R_DrawBrushModel`

**Contract** — draws a moving brush entity: rejects it by bounding sphere against the frustum, transforms the view
origin into the model's own space, marks the surfaces lit by nearby dynamic lights, then draws each surface facing the
viewer under the entity's transform.

**Invariants** — the view origin is moved into **model space rather than transforming the model**, the same decision as
in the software renderer and for the same reason: the back-face and plane tests then work on unmodified plane
equations.

Dynamic lights must be marked **inside** the model's space too, or a door lit by a rocket lights up in the wrong place.

## `R_DrawWaterSurfaces`, `R_MirrorChain`

**Contract** — draw the deferred liquid surfaces with the player's chosen transparency, and the mirror surfaces onto
the mirror chain. Fully opaque liquid takes a faster path that leaves the blend state alone.

**Invariants** — liquids are deferred to **after** all opaque geometry because they are blended, and blending requires
everything behind them to be present. That ordering requirement is the one thing about transparency that no renderer
escapes.

## `BuildSurfaceDisplayList`

**Contract** — called at load time per surface. Walks the surface's edge list into an explicit vertex list, computing
for each vertex its base texture coordinate from the surface's texture axes and its lightmap coordinate from the
surface's rectangle in its page. Optionally removes vertices that lie on the straight line between their neighbours.

```text
FUNCTION build_display_list(surf)
  collect the surface's vertices by walking its edges in order,
    following each edge forwards or backwards as the edge list's sign says
  FOR EACH vertex
    base texture coordinate = (vertex . axis + offset) / texture size
    lightmap coordinate     = (vertex . axis + offset - surface origin) / 16
                              shifted into the surface's page rectangle
                              and offset by half a sample
  IF removing shared edges
    drop any vertex whose two neighbours are nearly collinear with it
```

**Invariants** —

- The **half-sample offset** on the lightmap coordinate centres the texel over the sample. Omitting it shifts all
  lighting by half a lightmap texel, which looks like the lighting has slid off the geometry — a classic rebuild
  symptom.
- The edge list is walked **through the signed edge indices** of the map format ([`bspfile.h`](bspfile.h.md)): a
  negative index means traverse that edge backwards. That is how the compiler stores a shared edge once.
- **Collinear vertex removal is optional and off by default**, with a separate switch to report how many it would
  remove. It reduces vertex count but re-introduces cracks between adjacent surfaces on hardware with imprecise
  rasterization — which is why it defaults off and why the reporting switch exists. A rebuild on modern hardware can
  leave it on.

## `GL_DisableMultitexture`, `GL_EnableMultitexture`

**Contract** — turn the second texture unit on or off, doing nothing when the hardware has none.

**Invariants** — they are **stateful and must be paired**, because the unit left enabled changes how the next
unrelated draw is textured. Every path that enables must disable on every exit. In a rebuild this is the argument for
setting state per draw rather than maintaining it across draws.
