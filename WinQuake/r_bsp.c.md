# WinQuake/r_bsp.c

> The world tree walk: descends the rendering tree front to back, culling whole subtrees against the frustum, marking visible faces, and assigning each the sort key the span sorter orders by. Also clips a moving sub-model's faces against the world tree.

**Needs** — [`r_local.h`](r_local.h.md) · [`r_shared.h`](r_shared.h.md) · [`model.h`](model.h.md) · [`client.h`](client.h.md) · [`mathlib.h`](mathlib.h.md) · [`bspfile.h`](bspfile.h.md)
**Used by** — [`r_main.c`](r_main.c.md) drives it; [`d_edge.c`](d_edge.c.md) uses its sub-model rotation
**Tier floor** — none

## Purpose

Where the renderer decides what is visible and in what order. Three ideas.

**The walk is front to back and the order is the sort key.** Each node's faces are assigned an increasing
key as the walk reaches them, so a lower key means nearer — which is what makes the span sorter's
comparison a key comparison rather than a geometric one
([`r_edge.c`](r_edge.c.md#r_leadingedge)).

**Culling is hierarchical and self-simplifying.** A node is tested against only the frustum planes it might
still straddle, and a plane it is entirely inside is removed from the set passed to its children. So the
deeper the walk, the cheaper each test.

**A moving sub-model is clipped against the world tree**, so a door partly through a wall contributes only
its visible fragments. That is the alternative to depth-testing it, and it is why doors sort correctly with
no depth reads.

## State

```text
VARIABLE insubmodel : bool
VARIABLE currententity : Entity
VARIABLE modelorg, base_modelorg : vec3    # the view position in the CURRENT
                                           # entity's space
VARIABLE r_entorigin : vec3                # that entity's world position
VARIABLE entity_rotation : real[3][3]
VARIABLE r_worldmodelorg : vec3
VARIABLE r_currentbkey : int
CONSTANT max_bmodel_verts = 500 ;  max_bmodel_edges = 1000
VARIABLE pbverts, pbedges, numbverts, numbedges     # the clipping work pools
VARIABLE pfrontenter, pfrontexit : MVertex
VARIABLE makeclippededge : bool
```

**Invariants** — the view position is kept **in the current entity's space**, which is what lets the
backface test be a dot product against an untransformed plane. Drawing a sub-model replaces it and the base
copy is what restores it.

## `R_EntityRotate`, `R_RotateBmodel`

**Contract** — `R_RotateBmodel` builds the current entity's rotation from its three angles, rotates the view
basis and the view position into the entity's space, and re-derives the frustum.
`R_EntityRotate` applies that rotation to one vector.

**Invariants** — the matrix is built from three separate yaw, pitch and roll matrices multiplied together,
per call. The source lists four improvements it wants — a lookup table, storing the matrix with the entity,
caching lazily, and sharing the work with the model transform — and none was done. The cost is real
([`d_edge.c`](d_edge.c.md#d_drawsurfaces) calls it once per *surface*), and a rebuild should hoist it.

## `R_RecursiveWorldNode`

**Contract** — takes a node and the set of frustum planes it may straddle. Returns immediately for a solid
node or one not marked visible this frame. Culls against each remaining plane, dropping from the set any
plane the node is entirely inside. At a leaf, marks every face it references as visible, stores any entity
fragments, and assigns the leaf a sort key. At an interior node, recurses into the near child, draws that
node's faces, advances the key, then recurses into the far child.

```text
FUNCTION r_recursive_world_node(node, clipflags)
  IF node is solid                            RETURN
  IF node.visframe != r_visframecount          RETURN     # not potentially
                                                          # visible

  # --- hierarchical frustum cull ---
  FOR EACH frustum plane i STILL IN clipflags
    # Pick the box corner FURTHEST BEHIND the plane, from a precomputed index
    # triple; if even that is in front, the node is entirely outside.
    rejectpt = the node's box corner selected by pfrustum_indexes[i][0..2]
    IF dot(rejectpt, plane.normal) - plane.dist <= 0  RETURN   # fully outside
    # Pick the corner furthest IN FRONT; if that is behind, the node straddles.
    acceptpt = the corner selected by pfrustum_indexes[i][3..5]
    IF dot(acceptpt, plane.normal) - plane.dist >= 0
      clipflags = clipflags WITHOUT plane i     # entirely inside: children
                                               # need not test it

  IF node IS a leaf
    FOR EACH face the leaf references  mark it visible THIS frame
    IF the leaf has entity fragments  store them for drawing
    leaf.key = r_currentkey ;  r_currentkey = r_currentkey + 1
    RETURN

  # --- which side is the viewer on? ---
  dot = the signed distance FROM modelorg TO node.plane
        (one subtraction when the plane is axial, a dot product otherwise)
  side = 0 IF dot >= 0 ELSE 1

  r_recursive_world_node(node.children[side], clipflags)      # NEAR side first

  # --- this node's own faces ---
  IF node has faces
    FOR EACH face
      IF it faces the viewer                 # see the epsilon test below
         AND it was marked visible this frame
        r_render_face(face, clipflags)
    r_currentkey = r_currentkey + 1          # ALL faces on one node share a key

  r_recursive_world_node(node.children[NOT side], clipflags)  # FAR side
```

The backface test:

```text
# A face whose plane-back flag is set faces opposite its plane's normal.
IF dot < -backface_epsilon   draw faces WITH the plane-back flag
IF dot >  backface_epsilon   draw faces WITHOUT it
# Within the epsilon band, NEITHER is drawn.
```

**Invariants** — seven load-bearing decisions.

**The visibility stamp gate is what makes the walk cheap.** Only nodes on a path to a potentially-visible
leaf carry the current stamp ([`r_main.c`](r_main.c.md#r_markleaves)), so the walk never descends into an
unreachable half of the map.

**Faces are marked at the leaf and drawn at the node.** A face belongs to a node but is referenced by the
leaves that can see it, so the leaf pass marks and the node pass draws — and the node pass checks the mark.
That two-phase arrangement is what makes a face drawn once even though several leaves reference it.

**The clip-flag set shrinks as the walk descends.** A node entirely inside a plane removes it for its whole
subtree. That is the optimization, and it is why the flags are passed by value.

**The reject and accept corners are selected from a precomputed index triple per plane**, so choosing the
right corner of the box is three indexed loads rather than three comparisons. The indices are built from the
planes' sign bits at view-change time.

**The near child is recursed first and the node's own faces are drawn between the two recursions.** So the
key order is: everything nearer than this node, then this node's faces, then everything behind. That is
exactly back-to-front within each subtree and it is what the sorter relies on.

**All faces on one node share a sort key**, incremented once after the node. They are coplanar, so they
cannot occlude each other, and giving them one key lets the sorter's equal-key rule
([`r_edge.c`](r_edge.c.md#r_leadingedge)) resolve them by activation order.

**The backface epsilon of 0.01 rather than zero** means a face exactly edge-on is drawn by neither branch.
Without it a face at grazing incidence flickers between drawn and culled as the view moves by a fraction of
a unit.

**Notes** — the source marks the cull loop as compiling badly and estimates an assembly version at twice
the speed, and separately asks for integer sign-bit tests instead of floating-point comparisons. Neither was
done; the loop is the hot path of the walk.

The two disabled alternative paths — deferring polygons to a back-to-front list, or rendering them
immediately as polygons — serve the rasterizer capability flags
([`d_iface.h`](d_iface.h.md#the-capability-flags)) and do not run in this build.

## `R_RenderWorld`

**Contract** — sets the current entity to the world, the view position to the world position, and walks the
tree from its root with all four clip flags set. Then, if the rasterizer wanted deferred polygons, replays
them back to front.

**Invariants** — the deferred list is a 5000-entry stack array
([`r_local.h`](r_local.h.md)), about 40 kilobytes, allocated whether or not the path is taken.

## `R_DrawSubmodelPolygons`

**Contract** — draws every face of a sub-model that faces the viewer, without clipping against the world
tree. Each face takes its sort key from the leaf the sub-model's bounding box was found to be entirely
inside.

```text
FUNCTION r_draw_submodel_polygons(pmodel, clipflags)
  FOR EACH face IN the sub-model's slice of the surface array
    dot = the signed distance FROM modelorg TO the face's plane
    IF the face faces the viewer, by the same epsilon test as the world walk
      r_currentkey = the key OF the leaf the sub-model sits in
      r_render_face(face, clipflags)
```

**Invariants** — **every face of the sub-model gets the *same* key** — the containing leaf's — because the
sub-model occupies one leaf and therefore one depth slot in the world's ordering. Its faces cannot occlude
each other (it is convex enough in practice) but it can interpenetrate world geometry at that key, which is
what the sorter's coplanar depth tie-break exists for
([`r_edge.c`](r_edge.c.md#r_leadingedge)).

This is the **unclipped** path, used when the sub-model lies entirely within one leaf. The clipped path
below handles the other case.

## `R_DrawSolidClippedSubmodelPolygons`, `R_RecursiveClipBPoly`

**Contract** — draws a sub-model that spans several leaves, by recursively clipping each of its faces against
the world tree and emitting only the fragments that land in leaves the sub-model actually occupies. Each
fragment takes its leaf's key.

```text
FUNCTION r_recursive_clip_b_poly(edges, node, surf)
  # Split the face's edge loop by the node's plane, producing a front list and
  # a back list, adding a new edge along the plane where the loop crosses it.
  FOR EACH edge IN the loop
    classify both endpoints against the plane
    IF the edge crosses  compute the intersection and record it as a new
                         vertex, remembering which crossing ENTERS the front
                         and which EXITS
    append the edge, or its clipped part, TO the front list, the back list,
    or both
  IF the loop crossed the plane
    add an edge joining the entering and exiting intersections TO BOTH lists
  IF the front list is non-empty
    IF the front child IS a leaf  emit the fragment with that leaf's key
    ELSE recurse INTO the front child
  ...the same for the back list
```

**Invariants** — four things.

**A face is split, not rejected**, so a door half-way through a doorway contributes two fragments with
different sort keys — and each sorts correctly against the world geometry around it. That is the whole
reason this path exists.

**The new edge along the splitting plane must be added to both lists**, or the fragments are not closed
polygons and the edge sorter's span accounting breaks.

**The work pools are 500 vertices and 1000 edges**, fixed and stack-allocated. A sub-model complex enough to
exceed them silently truncates.

**The fragments use the leaf's key**, so a sub-model spanning three leaves occupies three depth slots — which
is exactly right.

**Notes** — this path is chosen by [`r_main.c`](r_main.c.md) when a sub-model's
bounding box does not fall entirely within one leaf, which the entity's recorded top node
([`render.h`](render.h.md)) records.
