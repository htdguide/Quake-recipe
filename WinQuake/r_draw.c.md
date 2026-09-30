# WinQuake/r_draw.c

> Turns a visible face into edges in the sorter's buckets: clips each edge against the frustum, projects it, and caches the result so a shared map edge is emitted once per frame.

**Needs** — [`r_local.h`](r_local.h.md) · [`r_shared.h`](r_shared.h.md) · [`model.h`](model.h.md) · [`d_iface.h`](d_iface.h.md) · [`mathlib.h`](mathlib.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops) ([`r_drawa.s`](r_drawa.s.md))
**Used by** — [`r_bsp.c`](r_bsp.c.md) calls the face renderers; [`r_edge.c`](r_edge.c.md) consumes the buckets
**Tier floor** — none

## Purpose

The bridge from geometry to the edge sorter. For each visible face: allocate a surface record, compute its
depth gradient, then walk its edge loop clipping each edge against the frustum, project the survivors, and
insert them into the per-scanline start and end buckets.

One decision dominates: **a map edge shared by two faces is clipped and projected once per frame**, and the
second face reuses the result. That halves the clipping work, and the caching mechanism is the file's most
intricate part.

## State

```text
CONSTANT maxleftclipedges = 100
CONSTANT fully_clipped_cached = 0x80000000      # a flag in the cache stamp
CONSTANT framecount_mask      = 0x7FFFFFFF
VARIABLE r_leftclipped, r_rightclipped : bool
VARIABLE r_leftenter, r_leftexit, r_rightenter, r_rightexit : MVertex
VARIABLE r_emitted, r_lastvertvalid, r_nearzi : int
VARIABLE r_u1, r_v1, r_lzi1, r_ceilv1 : real    # the previous vertex's
                                               # projection, cached
```

**Invariants** — the cache stamp is stored **in the map edge itself**
([`model.h`](model.h.md)'s cached offset field), packing a frame number and a fully-clipped flag into one
integer. So the cache lives in the geometry rather than in a side table, which is why it costs nothing to look
up.

## `R_EmitEdge`

**Contract** — takes two world-space vertices; projects both, rejects the edge if it spans no whole scanline,
computes its fixed-point x and per-scanline step, allocates an edge record, attaches the current surface to the
correct side, and links it into the start and end buckets for the scanlines it spans.

```text
FUNCTION r_emit_edge(pv0, pv1)
  # Project both endpoints, reusing the previous call's second endpoint when
  # this edge continues from it.
  IF NOT r_lastvertvalid
    project pv0: transform into view space, divide by depth, clamp to the view
    ceilv0 = ceil(v0)
  project pv1 likewise ;  ceilv1 = ceil(v1)

  IF ceilv0 == ceilv1  the edge spans no scanline ;  remember pv1 ;  RETURN

  IF ceilv0 < ceilv1                     # the edge goes DOWN the screen
    side = 0                             # it ENTERS the surface
    u_step = (u1 - u0) / (v1 - v0)
    u = u0 + (ceilv0 - v0) * u_step       # x at the first whole scanline
    v_top = ceilv0 ;  v_bottom = ceilv1 - 1
  ELSE                                   # the edge goes UP: swap and mark
    side = 1                             # it LEAVES the surface
    ...symmetrically

  IF v_bottom < v_top  RETURN
  edge = the next entry from the edge pool
  IF the pool is exhausted  r_outofedges = r_outofedges + 1 ;  RETURN
  edge.u = round_to_fixed_20(u) ;  edge.u_step = round_to_fixed_20(u_step)
  edge.surfs[side] = the current surface's index ;  the other side = 0
  edge.nearzi = the larger of the two depth reciprocals
  link edge INTO newedges[v_top], SORTED BY u
  link edge INTO removeedges[v_bottom]
```

**Invariants** — five things.

**An edge spanning no whole scanline is dropped.** The fill convention is that a scanline belongs to an edge
that crosses its *ceiling*, so an edge entirely between two scanlines contributes nothing. Getting this wrong
produces one-pixel gaps between adjacent faces.

**The direction decides which of the two surface slots is filled.** A downward edge enters the surface and an
upward one leaves it ([`r_shared.h`](r_shared.h.md)), which is what lets the sorter push and pop rather than
re-sort.

**The removal bucket is indexed by the last scanline the edge occupies**, one less than the ceiling of its
lower endpoint — again the fill convention.

**The new-edge bucket is kept sorted by x on insertion**, so the sorter's merge is linear
([`r_edge.c`](r_edge.c.md#r_insertnewedges)).

**Pool exhaustion increments a counter and drops the edge**, which the frame reports
([`r_main.c`](r_main.c.md)). The face is then drawn with a hole.

The previous vertex's projection is cached in file-level variables so that walking an edge loop projects each
vertex once rather than twice.

## `R_ClipEdge`

**Contract** — takes two world-space vertices and a chain of clip planes; recursively clips the edge against
each plane in turn and emits whatever survives. Records where the edge entered and exited the left and right
frustum planes, so the caller can close the polygon along the screen edge.

```text
FUNCTION r_clip_edge(pv0, pv1, clip)
  WHILE clip EXISTS
    d0 = the signed distance FROM pv0 TO clip's plane
    d1 = the same FOR pv1
    IF d0 >= 0                                   # pv0 is inside
      IF d1 >= 0  clip = clip.next ;  CONTINUE    # wholly inside this plane
      # Crosses outward: clip pv1 to the plane.
      new = the intersection
      IF clip names a SCREEN edge  record it as the EXIT point
      pv1 = new
    ELSE                                          # pv0 is outside
      IF d1 < 0  RETURN                           # wholly outside: reject
      new = the intersection
      IF clip names a screen edge  record it as the ENTER point
      pv0 = new
    clip = clip.next
  r_emit_edge(pv0, pv1)
```

**Invariants** — three things.

**The enter and exit points on the left and right planes are recorded**, and the caller uses them to add an
edge along the screen boundary — which closes a polygon that the frustum cut open. Without it the sorter sees
an unclosed loop and its span accounting breaks.

**Clipping is against the *world-space* frustum planes**, which is why the frustum is transformed
([`r_misc.c`](r_misc.c.md#r_transformfrustum-transformvector-r_transformplane)) rather than the geometry.

**The clip epsilon is 0.001** ([`r_local.h`](r_local.h.md)), applied so a vertex exactly on a plane is treated
as inside — which stops a shared vertex between two faces being clipped differently in each.

## `R_EmitCachedEdge`

**Contract** — re-emits an edge already clipped and projected this frame, attaching the current surface to the
side the first emission left empty.

```text
FUNCTION r_emit_cached_edge()
  edge = the edge the map edge's cache stamp names
  IF edge.surfs[0] IS unset  edge.surfs[0] = the current surface
  ELSE                       edge.surfs[1] = the current surface
  r_emitted = 1
```

**Invariants** — this is the payoff of the caching. The first face to reach a shared map edge clips, projects
and emits it, filling one surface slot; the second face finds the stamp and merely fills the other slot. So the
edge exists once with both its surfaces, which is exactly what the sorter needs.

The cache stamp packs the frame number and a fully-clipped flag
([`model.h`](model.h.md)), so an edge clipped entirely away is remembered as such and the second face does not
retry it.

## `R_RenderFace`

**Contract** — takes a face and the clip flags its node inherited. Allocates a surface record, rejects the face
if the pool is exhausted, computes its depth gradient from its plane, then walks its edge loop — emitting each
edge from the cache when possible and clipping it otherwise — and finally closes the polygon along the left and
right screen edges if the frustum cut it.

```text
FUNCTION r_render_face(fa, clipflags)
  # Build the clip-plane chain from the flags still set.
  pclip = a chain of the view clip planes named by clipflags
  surf = the next surface from the pool
  IF exhausted  r_outofsurfaces = r_outofsurfaces + 1 ;  RETURN

  # The depth gradient, from the face's plane in view space.
  transform the face's plane into view space
  surf.d_ziorigin, surf.d_zistepu, surf.d_zistepv = the gradient
  surf.key = r_currentkey ;  surf.data = fa ;  surf.flags = fa.flags
  surf.insubmodel = insubmodel ;  surf.spanstate = 0 ;  surf.spans = nothing
  surf.last_u = 0 ;  surf.nearzi = the largest 1/z on the face

  r_leftclipped = r_rightclipped = false ;  r_emitted = 0
  r_lastvertvalid = false
  FOR EACH of the face's edges, in winding order
    v0, v1 = its two vertices, honouring the negative-index convention
    IF the map edge's cache stamp IS this frame
      IF it was fully clipped  CONTINUE
      r_emit_cached_edge()
    ELSE
      r_clip_edge(v0, v1, pclip)
      stamp the map edge with this frame and whether it survived
    r_lastvertvalid = true

  IF nothing was emitted  release the surface ;  RETURN
  IF the left plane clipped the face   emit an edge FROM its enter TO its exit
  IF the right plane clipped the face  emit an edge likewise
```

**Invariants** — four things.

**The depth gradient is computed per face, from its plane**, and stored in the surface record — which is what
the sorter compares ([`r_edge.c`](r_edge.c.md#r_leadingedge)) and what the rasterizer steps
([`d_edge.c`](d_edge.c.md)).

**A face that emitted no edges releases its surface record**, so the pool is not consumed by invisible faces.

**The screen-edge closing edges are emitted last**, after every real edge, which is why the enter and exit
points must be recorded rather than handled inline.

**The nearest depth reciprocal on the face is recorded** and is what the rasterizer's mip selection uses
([`d_edge.c`](d_edge.c.md#d_drawsurfaces)) — so one mip level serves the whole face.

The source carries seven notes asking for caching, screen-space clipping, and shared clipped edges. All are
real optimizations and none changes behaviour.

## `R_RenderBmodelFace`

**Contract** — as the face renderer, but for a face already clipped into fragments by the world tree
([`r_bsp.c`](r_bsp.c.md#r_drawsolidclippedsubmodelpolygons-r_recursiveclipbpoly)): walks a fragment's own edge list rather than the map's, with no
edge caching.

**Invariants** — no caching, because a clipped fragment's edges are not shared. The source notes it would like
the fragment's edges kept in a consistent winding so that last-vertex caching would work.

## `R_RenderPoly`

**Contract** — the alternative path for a rasterizer that wants clipped polygons instead of edges
([`d_iface.h`](d_iface.h.md)): clips the face to a vertex list and hands it over. Not used in this build.

**Notes** — the source flags it as needing cleanup and as lacking a vertex-count guard.

## `R_ZDrawSubmodelPolys`

**Contract** — draws a sub-model's faces into the **depth buffer only**, for a sub-model that must occlude
without being drawn.

**Invariants** — used for the transparent-brush case; writes depth so that things behind it are hidden while
the brush itself is not shaded.
