# WinQuake/world.c

> Collision: turns every bounding box into a six-plane binary tree so that one recursive sweep handles both map geometry and entities, indexed by a four-level spatial subdivision.

**Needs** — [`world.h`](world.h.md) · [`quakedef.h`](quakedef.h.md) · [`bspfile.h`](bspfile.h.md) · [`model.h`](model.h.md) (the loaded hulls) · [`progs.h`](progs.h.md) · [`server.h`](server.h.md) · [`common.h`](common.h.md) (the intrusive list) · [`mathlib.h`](mathlib.h.md) · [`pr_exec.c`](pr_exec.c.md) (trigger callbacks) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops) (the point-contents walk has an assembly twin in [`worlda.s`](worlda.s.md))
**Used by** — [`sv_phys.c`](sv_phys.c.md) · [`sv_move.c`](sv_move.c.md) · [`sv_user.c`](sv_user.c.md) · [`pr_cmds.c`](pr_cmds.c.md) · [`sv_main.c`](sv_main.c.md) · [`pr_edict.c`](pr_edict.c.md)
**Tier floor** — none; the recursion and the arithmetic are tier-free

## Purpose

This file answers one question — "sweeping this box from here to there, what does it hit
first" — and the way it answers is the engine's cleverest single idea.

The map's collision geometry is a binary tree of planes ([`bspfile.h`](bspfile.h.md)), so
sweeping against the map is a recursive descent. An entity's collision geometry is a
bounding box, which is a different shape entirely. Rather than write a second algorithm,
the file **fabricates a six-node binary tree whose leaves are exactly a box** and reuses
the same descent. So there is one sweep routine, one impact-point computation, one epsilon,
and one set of numerical quirks, applied uniformly to walls and to monsters.

The second idea is the spatial index: a four-level subdivision of the map into 31 nodes,
with entities linked into the smallest node that wholly contains them, and separate lists
for solid and trigger entities at each node. A sweep visits only the nodes its path
crosses.

## State

```text
# --- the fabricated box tree, ONE instance, reused ---
VARIABLE box_hull      : Hull
VARIABLE box_clipnodes : ClipNode[6]
VARIABLE box_planes    : Plane[6]

# --- the spatial index ---
RECORD AreaNode
  axis     : int            # 0 or 1, or -1 for a leaf
  dist     : real           # the splitting coordinate
  children : AreaNode[2]
  trigger_edicts : Link     # list header
  solid_edicts   : Link     # list header

CONSTANT area_depth = 4
CONSTANT area_nodes = 32
VARIABLE sv_areanodes   : AreaNode[32]
VARIABLE sv_numareanodes : int

# --- one sweep in progress ---
RECORD MoveClip
  boxmins, boxmaxs : vec3   # the whole swept volume, for broad-phase rejection
  mins, maxs       : vec3   # the moving object's box
  mins2, maxs2     : vec3   # its box WHEN TESTED AGAINST MONSTERS
  start, end       : vec3
  trace            : Trace  # the best result so far
  type             : int    # the trace mode
  passedict        : optional<Edict>

CONSTANT dist_epsilon = 0.03125         # 1/32 of a world unit
```

**Invariants** — the box tree is a **single static instance**, rewritten on every use. So
only one box-versus-box test can be in progress at a time, and the sweep against entities
is strictly sequential. A rebuild that parallelizes collision must give each worker its own.

The spatial index splits only on the two **horizontal** axes, never vertically. That is a
decision about the shape of the maps: they are broad and shallow, and a vertical split would
put most of the world in one child.

## The fabricated box tree

### `SV_InitBoxHull`

**Contract** — builds the six-node tree's topology once at level load. Node *i* splits on
plane *i*; plane *i* is axis-aligned along axis `i/2`; the children are arranged so that
descending toward the box's interior walks nodes 0 through 5 and arriving past node 5 means
"inside", while stepping off any node the other way means "outside".

```text
FUNCTION sv_init_box_hull()
  box_hull spans clipnodes 0 through 5
  FOR i FROM 0 TO 5
    box_clipnodes[i].planenum = i
    side = i BITAND 1                      # alternate which child is "outside"
    box_clipnodes[i].children[side]     = CONTENTS_EMPTY
    box_clipnodes[i].children[side XOR 1] = (i + 1) IF i != 5 ELSE CONTENTS_SOLID
    box_planes[i].type = i SHIFTED RIGHT 1      # axis 0, 0, 1, 1, 2, 2
    box_planes[i].normal[i SHIFTED RIGHT 1] = 1 # all normals point +axis
```

**Invariants** — all six normals point in the **positive** axis direction; the alternation
of which child is empty is what makes the odd-numbered planes act as minimum bounds despite
their positive normals. Concretely: plane 0 is `x = maxs[0]` and being in front of it is
outside; plane 1 is `x = mins[0]` and being *behind* it is outside.

The tree is fully determined by its topology, so only the six plane distances change per
use — which is the whole point.

### `SV_HullForBox`

**Contract** — takes a minimum and maximum corner; writes the six distances and returns
the shared tree.

```text
FUNCTION sv_hull_for_box(mins, maxs) -> Hull
  box_planes[0].dist = maxs[0] ;  box_planes[1].dist = mins[0]
  box_planes[2].dist = maxs[1] ;  box_planes[3].dist = mins[1]
  box_planes[4].dist = maxs[2] ;  box_planes[5].dist = mins[2]
  RETURN box_hull
```

**Notes** — the source's comment says it best: so that six floats can be stored out and
become a proper tree. A rebuild that writes a separate box sweep loses the uniformity and
gains a second set of epsilon bugs.

### `SV_HullForEntity`

**Contract** — takes an entity and the moving object's box; returns the tree to sweep
against and the offset to subtract from the sweep's endpoints to bring them into that
tree's frame. A map-geometry entity contributes one of its three precompiled hulls, chosen
by the moving object's width; any other entity contributes a fabricated box **expanded by
the mover's size**. A geometry entity that is not a pushing entity, or whose model is not
map geometry, is a fatal error.

```text
FUNCTION sv_hull_for_entity(ent, mins, maxs) -> (Hull, offset)
  IF ent.solid IS solid_bsp
    IF ent.movetype IS NOT push  FAIL WITH "SOLID_BSP without MOVETYPE_PUSH"
    model = sv.models[ent.modelindex]
    IF model IS nothing OR model is not map geometry
      FAIL WITH "MOVETYPE_PUSH with a non bsp model"
    size = maxs - mins
    hull = model.hulls[0] IF size[0] <  3        # a point
           model.hulls[1] IF size[0] <= 32       # a player or small monster
           model.hulls[2] OTHERWISE              # a large monster
    # the compiled hull was shrunk by ITS OWN box, so shift by the difference
    offset = (hull.clip_mins - mins) + ent.origin
  ELSE
    # MINKOWSKI SUM: grow the target's box by the mover's, so the mover
    # becomes a point
    hullmins = ent.mins - maxs
    hullmaxs = ent.maxs - mins
    hull = sv_hull_for_box(hullmins, hullmaxs)
    offset = ent.origin
  RETURN (hull, offset)
```

**Invariants** — the expansion for the non-geometry case is a **Minkowski sum**: subtracting
the mover's maximum from the target's minimum and the mover's minimum from the target's
maximum turns a box-versus-box sweep into a point-versus-box sweep. That is why every trace
in the engine ultimately traces a point.

For map geometry the same trick was done *offline*: the map compiler built three trees, each
already shrunk by one of three fixed body sizes ([`bspfile.h`](bspfile.h.md)). The
**hull selection is by the mover's width alone** — under 3 units is a point, up to 32 is
the player hull, above that the large hull. Those thresholds and the three hulls' box sizes
are an unwritten contract with the map compiler; they live in
[`model.c`](model.c.md#mod_loadbrushmodel) and must agree.

Because the compiled hull is shrunk by *its* box rather than by the mover's, the offset has
to correct for the difference. A rebuild that gets that correction wrong produces collision
that is off by a few units — enough to make players stick in doorways.

---

## The spatial index

### `SV_CreateAreaNode`

**Contract** — recursively subdivides a volume four levels deep, splitting on whichever
horizontal axis is longer, at the midpoint. Each node carries two empty list headers.

```text
FUNCTION sv_create_area_node(depth, mins, maxs) -> AreaNode
  anode = the next free slot
  clear both list headers
  IF depth == 4
    anode.axis = -1 ;  no children ;  RETURN anode
  size = maxs - mins
  anode.axis = 0 IF size[0] > size[1] ELSE 1        # horizontal axes only
  anode.dist = midpoint OF the chosen axis
  anode.children[0] = the FAR half   (from dist to maxs)
  anode.children[1] = the NEAR half  (from mins to dist)
  RETURN anode
```

**Invariants** — child 0 is the half **above** the split and child 1 the half below, which
every descent in this file relies on. Depth 4 gives 16 leaves and 31 nodes total; the array
holds 32.

**Notes** — the recursion builds children before returning, so the array is filled
breadth-then-depth in a fixed order. Nothing depends on the order.

### `SV_ClearWorld`

**Contract** — builds the box tree topology and the spatial index over the world model's
bounds. Called once per level.

### `SV_UnlinkEdict`

**Contract** — removes an entity from whichever list holds it and marks it unlinked. Safe
on an entity that is not linked.

### `SV_FindTouchedLeafs`

**Contract** — walks the *rendering* tree recording which leaves an entity's box overlaps,
up to sixteen, stopping early if it exceeds that. Solid nodes are not entered.

```text
FUNCTION sv_find_touched_leafs(ent, node)
  IF node is solid  RETURN
  IF node IS a leaf
    IF ent.num_leafs == 16  RETURN                # too many; give up
    ent.leafnums[ent.num_leafs] = index_of(leaf) - 1
    ent.num_leafs = ent.num_leafs + 1
    RETURN
  sides = box_on_plane_side(ent.absmin, ent.absmax, node.plane)
  IF sides HAS the front bit  recurse INTO node.children[0]
  IF sides HAS the back bit   recurse INTO node.children[1]
```

**Invariants** — this walks the **rendering** tree, not a collision hull, because its
purpose is visibility: the leaf list is what the server uses to decide whether a client can
see this entity ([`sv_main.c`](sv_main.c.md)). The leaf index is offset by one because leaf
0 is the solid leaf and carries no visibility.

Stopping at sixteen leaves the count *at* sixteen, which downstream reads as "spans too
many leaves" and treats the entity as always visible. A rebuild that silently truncates
instead makes large entities disappear from parts of the map.

### `SV_LinkEdict`

**Contract** — recomputes an entity's world-space box, expands it, recomputes its leaf list,
and links it into the smallest index node that contains it — into the trigger list if it is
a trigger, otherwise the solid list. The world entity and free entities are not linked.
Non-solid entities get a leaf list but no index link. When asked, then runs every
overlapping trigger's touch callback.

```text
FUNCTION sv_link_edict(ent, touch_triggers)
  IF ent is already linked  sv_unlink_edict(ent)
  IF ent IS the world  RETURN
  IF ent IS free       RETURN

  ent.absmin = ent.origin + ent.mins
  ent.absmax = ent.origin + ent.maxs

  IF ent.flags HAS the item bit
    expand absmin/absmax by 15 units HORIZONTALLY only
  ELSE
    expand absmin/absmax by 1 unit on EVERY axis

  ent.num_leafs = 0
  IF ent.modelindex != 0
    sv_find_touched_leafs(ent, the world model's root)

  IF ent.solid IS not-solid  RETURN

  # descend to the smallest node wholly containing the box
  node = the index root
  LOOP
    IF node.axis == -1                        BREAK   # a leaf
    IF ent.absmin[node.axis] > node.dist      node = node.children[0]
    ELSE IF ent.absmax[node.axis] < node.dist node = node.children[1]
    ELSE                                      BREAK   # straddles; stop here

  link ent INTO node.trigger_edicts IF ent.solid IS trigger
                ELSE node.solid_edicts

  IF touch_triggers  sv_touch_links(ent, the index root)
```

**Invariants** — two different expansions, and both are load-bearing.

The **15-unit horizontal** expansion for items is documented in the source: it makes items
easier to pick up and lets them be grabbed off shelves. It is a gameplay decision baked
into the collision system, applied to anything the game flags as an item, and it explains
why a player collects a box of shells by walking near it rather than into it. It is
horizontal only, so a player cannot collect an item from a floor above.

The **1-unit** expansion on everything else exists because movement stops an epsilon short
of a surface, so two boxes that "should" touch may be a fraction apart; without the
expansion, the overlap test would miss contacts the physics created.

An entity straddling a split stays at that node, so **a node's list holds entities larger
than its children** and a sweep must test every node along its path, not just the leaf it
ends in.

An entity with no model gets an empty leaf list, which downstream reads as "nowhere", so a
modelless entity is never sent to clients — which is how the game hides its bookkeeping
entities.

### `SV_TouchLinks`

**Contract** — takes an entity and an index node; for every trigger in that node's trigger
list whose box overlaps the entity's, runs that trigger's touch callback with the trigger as
the acting entity and the mover as the other. Recurses into whichever children the entity's
box reaches. Saves and restores the interpreter's acting-entity globals around each
callback.

```text
FUNCTION sv_touch_links(ent, node)
  FOR EACH touch IN node.trigger_edicts        # next link cached FIRST:
                                               # the callback may unlink it
    IF touch IS ent                              CONTINUE
    IF touch has no touch callback               CONTINUE
    IF touch.solid IS NOT trigger                CONTINUE
    IF the two world-space boxes do not overlap  CONTINUE

    saved_self = the global self ;  saved_other = the global other
    the global self  = touch
    the global other = ent
    the global time  = sv.time
    run touch.touch to completion               # arbitrary game code
    the global self = saved_self ;  the global other = saved_other

  IF node.axis == -1  RETURN
  IF ent.absmax[node.axis] > node.dist  recurse INTO node.children[0]
  IF ent.absmin[node.axis] < node.dist  recurse INTO node.children[1]
```

**Invariants** — the next list link is captured **before** the callback runs, because a
trigger's callback commonly removes the trigger or the mover, which unlinks a node the loop
is standing on. This is the single most important detail in the function.

The acting-entity convention — the trigger is `self`, the mover is `other` — is the game's
universal contract for touch callbacks, and every touch function in
[`qw-qc/`](../qw-qc/README.md) is written against it.

**Notes** — a callback may also *link* new entities, which inserts into lists the walk has
not reached. The original tolerates it; a newly spawned trigger may or may not fire this
frame depending on where it lands.

---

## Point queries

### `SV_HullPointContents`

**Contract** — takes a collision tree and a starting node and a point; descends until it
reaches a contents value and returns it. A node index outside the tree's range is a fatal
error.

```text
FUNCTION sv_hull_point_contents(hull, num, p) -> int
  WHILE num >= 0                              # negative values are contents
    IF num outside hull's clipnode range  FAIL WITH "bad node number"
    node  = hull.clipnodes[num]
    plane = hull.planes[node.planenum]
    d = (p[plane.type] - plane.dist)          IF plane.type < 3    # axial
        (dot(plane.normal, p) - plane.dist)   OTHERWISE
    num = node.children[1] IF d < 0 ELSE node.children[0]
  RETURN num
```

**Invariants** — child 0 is the front side and child 1 the back, and a point exactly on a
plane goes **front**. The axial shortcut is the same one the box-side predicate uses
([`mathlib.h`](mathlib.h.md)) and matters because three quarters of planes are axial.

**Notes** — this routine has an assembly twin ([`worlda.s`](worlda.s.md)) selected at
compile time; the contract is identical.

### `SV_PointContents`, `SV_TruePointContents`

**Contract** — evaluate the world's collision tree at a point. The plain form collapses the
six directional-current contents values into plain water; the true form returns them
unchanged.

```text
FUNCTION sv_point_contents(p) -> int
  c = sv_hull_point_contents(world hull 0, root, p)
  IF c IS one OF the six current contents  RETURN water
  RETURN c
```

**Invariants** — the collapse works by a *range* test on the contents values, which is why
their numbering in [`bspfile.h`](bspfile.h.md) must stay contiguous.

### `SV_TestEntityPosition`

**Contract** — takes an entity; sweeps its box from its own position to its own position and
reports whether the sweep began inside solid. Returns the world entity when stuck and
nothing otherwise.

**Notes** — returning the *world* as a truthy "yes, stuck" rather than what it is stuck in
is a wart; callers only test for truth. The source's own comment notes the whole approach
could be more efficient — it performs a full sweep to answer a point question.

---

## The sweep

### `SV_RecursiveHullCheck`

**Contract** — sweeps a point along a segment through a collision tree, filling in the
trace. Returns whether the sweep completed without impact. Splits the segment at each plane
it crosses, recurses into the near side first, and on finding the far side solid records the
impact fraction, position and plane. A node index outside the tree's range is a fatal error.

```text
FUNCTION sv_recursive_hull_check(hull, num, p1f, p2f, p1, p2, trace) -> bool
  # p1f and p2f are the segment's endpoints as fractions of the ORIGINAL sweep.

  IF num < 0                                  # a leaf: `num` IS a contents value
    IF num IS NOT solid
      trace.allsolid = false
      trace.inopen = true   IF num IS empty
      trace.inwater = true  OTHERWISE
    ELSE
      trace.startsolid = true
    RETURN true                               # this span is passable

  IF num outside hull's clipnode range  FAIL WITH "bad node number"
  node  = hull.clipnodes[num]
  plane = hull.planes[node.planenum]
  t1, t2 = signed distances OF p1, p2 FROM plane   # axial shortcut as above

  IF t1 >= 0 AND t2 >= 0  RETURN recurse INTO children[0] OVER THE WHOLE segment
  IF t1 <  0 AND t2 <  0  RETURN recurse INTO children[1] OVER THE WHOLE segment

  # The segment crosses the plane. Put the crossing point an epsilon back
  # toward the NEAR side, so the split point is never exactly on the plane.
  frac = (t1 + dist_epsilon) / (t1 - t2)   IF t1 < 0
         (t1 - dist_epsilon) / (t1 - t2)   OTHERWISE
  clamp frac INTO [0, 1]
  midf = p1f + (p2f - p1f) * frac
  mid  = p1 + (p2 - p1) * frac
  side = 1 IF t1 < 0 ELSE 0                  # which child holds p1

  # Near half first.
  IF NOT recurse INTO children[side] OVER (p1f..midf, p1..mid)
    RETURN false                             # it hit something nearer

  # Near half was clear. Is the far half solid?
  IF the far child's contents at `mid` IS NOT solid
    RETURN recurse INTO children[side XOR 1] OVER (midf..p2f, mid..p2)

  IF trace.allsolid  RETURN false             # never left solid at all

  # --- impact ---
  IF side == 0
    trace.plane = the plane as given
  ELSE
    trace.plane = the plane NEGATED            # face the mover
  # `mid` should be outside solid, but occasionally is not; back off until it is
  WHILE the WHOLE hull's contents at `mid` IS solid
    frac = frac - 0.1
    IF frac < 0
      trace.fraction = midf ;  trace.endpos = mid
      print "backup past 0"
      RETURN false
    midf = p1f + (p2f - p1f) * frac
    mid  = p1 + (p2 - p1) * frac

  trace.fraction = midf
  trace.endpos = mid
  RETURN false
```

**Invariants** — this is the most numerically delicate routine in the engine and every
detail below is load-bearing.

The **epsilon is 1/32 of a world unit** and is applied *toward the near side* when splitting.
So the recorded impact point is always a thirty-second of a unit short of the surface, never
on it and never past it. That gap is why [`SV_LinkEdict`](#sv_linkedict) has to expand
bounding boxes by a unit: entities end up not quite touching.

The **near side is recursed first and a hit there ends the sweep**, which is what makes the
result the *first* impact rather than any impact. The "did the far child turn solid" test is
a point query at the split point, not another sweep, and it is what distinguishes "the
segment continues into open space" from "the segment has hit a wall".

The **plane is negated when the mover approached from behind**, so the returned normal always
faces the mover. Physics relies on that: it projects velocity onto the normal, and a
normal facing the wrong way would push the mover into the wall.

The **back-off loop** is the interesting confession. After computing an impact point an
epsilon short of the surface, the code checks whether that point is nonetheless inside solid
— which can happen where several planes meet at a sharp angle and the epsilon along one
plane pushes the point through another. It then walks the point back toward the start in
tenths of the crossing fraction, up to ten times, and if it still cannot find open space it
reports failure with a diagnostic. The source's comment says it shouldn't really happen but
does occasionally. A rebuild must implement it: without the back-off, entities get
permanently stuck in map corners, which is exactly the class of bug this loop exists to
avoid.

**Notes** — the file contains a commented-out alternative for the both-sides-same-sign test
that uses the epsilon and an ordering condition. It is the more careful version and it is
*not* what ships; a rebuild should match the shipping form, because the epsilon behaviour
at grazing angles differs and published maps have been played against the shipping one.

### `SV_ClipMoveToEntity`

**Contract** — sweeps a box against one entity and returns the trace. Selects the entity's
collision tree, moves the sweep into that tree's frame, performs the sweep, moves the result
back, and attributes the hit to the entity if there was one.

```text
FUNCTION sv_clip_move_to_entity(ent, start, mins, maxs, end) -> Trace
  trace = { fraction: 1, allsolid: true, endpos: end }   # the optimistic default
  hull, offset = sv_hull_for_entity(ent, mins, maxs)
  start_local = start - offset
  end_local   = end   - offset
  sv_recursive_hull_check(hull, hull's root, 0, 1, start_local, end_local, trace)
  IF trace.fraction != 1  trace.endpos = trace.endpos + offset
  IF trace.fraction < 1 OR trace.startsolid  trace.ent = ent
  RETURN trace
```

**Invariants** — `allsolid` starts **true** and is cleared by the sweep on reaching any
non-solid leaf. So "all solid" is the default and the sweep's job is to disprove it. A
rebuild that initializes it false reports clean moves through walls.

The endpoint is un-offset only when there *was* an impact, because otherwise it already
holds the caller's world-space destination.

**Notes** — the sequel's rotating-geometry support rotates the endpoints into the entity's
frame before the sweep and rotates the result back after; it is compiled out here. That
code is where a rebuild wanting rotating brush entities should start, and it is worth
noting that the original's build has no rotation at all — a rotated door collides as its
unrotated self.

### `SV_ClipToLinks`

**Contract** — takes an index node and a sweep in progress; sweeps against every solid
entity in that node whose box the swept volume reaches, keeping the nearest result. Recurses
into whichever children the swept volume reaches. Skips non-solid entities, the excluded
entity, anything the mode excludes, anything whose box misses the swept volume, points
against points, and the owner relationships. A trigger found in a solid list is a fatal
error.

```text
FUNCTION sv_clip_to_links(node, clip)
  FOR EACH touch IN node.solid_edicts           # next link cached first
    IF touch.solid IS not-solid                          CONTINUE
    IF touch IS clip.passedict                           CONTINUE
    IF touch.solid IS trigger  FAIL WITH "Trigger in clipping list"
    IF clip.type IS nomonsters AND touch.solid IS NOT solid_bsp  CONTINUE
    IF touch's box does not overlap clip.boxmins..boxmaxs        CONTINUE
    IF clip.passedict has size AND touch has NO size             CONTINUE
                                                  # points never interact
    IF clip.trace.allsolid  RETURN                # already hopeless
    IF clip.passedict EXISTS
      IF touch.owner IS clip.passedict            CONTINUE  # own missiles
      IF clip.passedict.owner IS touch            CONTINUE  # own owner

    # monsters are tested against the possibly-enlarged box
    box = (clip.mins2, clip.maxs2) IF touch.flags HAS the monster bit
          ELSE (clip.mins, clip.maxs)
    trace = sv_clip_move_to_entity(touch, clip.start, box, clip.end)

    IF trace.allsolid OR trace.startsolid OR trace.fraction < clip.trace.fraction
      trace.ent = touch
      IF clip.trace.startsolid
        clip.trace = trace ;  clip.trace.startsolid = true   # preserve the flag
      ELSE
        clip.trace = trace
    ELSE IF trace.startsolid
      clip.trace.startsolid = true

  IF node.axis == -1  RETURN
  IF clip.boxmaxs[node.axis] > node.dist  recurse INTO node.children[0]
  IF clip.boxmins[node.axis] < node.dist  recurse INTO node.children[1]
```

**Invariants** — the two **owner exclusions** are the reason a rocket does not hit the
player who fired it and does not hit a second rocket from the same player. They are checked
in both directions, which also means a player cannot be blocked by their own projectile.

**"Points never interact"** excludes the case where the mover has size and the target does
not; a zero-sized entity is a marker and is never an obstacle.

The **startsolid flag is preserved across results**. Once any entity reports that the sweep
began inside it, that fact must survive being superseded by a nearer impact, because physics
uses it to decide whether the mover needs extracting.

The **enlarged monster box** is where the missile mode's inflation is applied, and only to
entities the game has flagged as monsters — so a rocket is generous against a monster and
exact against a wall or a door.

The early return when the sweep is already entirely solid is an optimization that also
changes behaviour: entities after that point in the list are not consulted at all, so the
attributed entity depends on list order.

**Notes** — the trigger-in-solid-list check is an assertion about
[`SV_LinkEdict`](#sv_linkedict)'s correctness, not about input.

### `SV_MoveBounds`

**Contract** — computes the axis-aligned volume the whole sweep occupies, expanded by one
unit on each side.

```text
FUNCTION sv_move_bounds(start, mins, maxs, end) -> (boxmins, boxmaxs)
  FOR EACH axis i
    IF end[i] > start[i]
      boxmins[i] = start[i] + mins[i] - 1
      boxmaxs[i] = end[i]   + maxs[i] + 1
    ELSE
      boxmins[i] = end[i]   + mins[i] - 1
      boxmaxs[i] = start[i] + maxs[i] + 1
```

**Invariants** — this is the broad-phase rejection volume, so it must be conservative; the
one-unit margin matches the expansion in [`SV_LinkEdict`](#sv_linkedict).

**Notes** — the file contains a commented-out variant that returns an enormous box, so that
every entity is tested. A debugging switch, and a useful one for a rebuild validating its
broad phase.

### `SV_Move`

**Contract** — the public sweep. Clips against the world first, then against every relevant
entity, and returns the nearest result.

```text
FUNCTION sv_move(start, mins, maxs, end, type, passedict) -> Trace
  clip = all zero
  clip.trace = sv_clip_move_to_entity(the world entity, start, mins, maxs, end)
  clip.start = start ;  clip.end = end
  clip.mins  = mins  ;  clip.maxs = maxs
  clip.type  = type  ;  clip.passedict = passedict

  IF type IS move_missile
    clip.mins2 = (-15, -15, -15) ;  clip.maxs2 = (15, 15, 15)
  ELSE
    clip.mins2 = mins ;  clip.maxs2 = maxs

  clip.boxmins, clip.boxmaxs = sv_move_bounds(start, clip.mins2, clip.maxs2, end)
  sv_clip_to_links(the index root, clip)
  RETURN clip.trace
```

**Invariants** — the world is clipped **first and unconditionally**, and its result seeds
the sweep. So an entity can only shorten the result, never lengthen it, and the world is
never excluded by the passed entity or the mode.

The broad-phase volume is computed from the **enlarged** box, so in missile mode the volume
is large enough to find the monsters the enlargement is meant to catch.

The missile enlargement is a hard-coded 30-unit cube. It is not derived from anything and
a rebuild should keep the number.
