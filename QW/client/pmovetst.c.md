# QW/client/pmovetst.c

> The collision queries the movement model needs, answered against the map's precompiled hulls and against a fabricated six-node box tree, so that client and server reach identical conclusions.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`pmove.h`](pmove.h.md) · [`model.h`](model.h.md) · [`mathlib.h`](mathlib.h.md)
**Used by** — [`pmove.c`](pmove.c.md)
**Tier floor** — none

## Purpose

The other half of the movement extraction. [`pmove.c`](pmove.c.md) holds the physics; this holds the three collision
queries it makes, reimplemented over the flat world list in [`pmove.h`](pmove.h.md) rather than over the server's entity
tree.

Read [`world.c`](../server/world.c.md) first: this is the same collision model — the same three queries, the same
Minkowski-sum reduction of a box sweep to a point sweep, the same fabricated box hull, the same epsilon — with the entity
lookup replaced. The differences are what this twin records, and the fact that **a client can answer these questions at all**
is what makes prediction possible.

## State

```text
VARIABLE box_hull   : a 6-node hull, rebuilt per query
VARIABLE box_clipnodes : node[6] ;  box_planes : plane[6]
CONSTANT the epsilon the sweep backs off by
```

## `PM_InitBoxHull`, `PM_HullForBox`

**Contract** — build, once, a six-node tree whose planes are the six faces of an axis-aligned box: node *i* tests plane *i*,
its outside child is empty space, its inside child is the next node, and the last node's inside child is solid. Then, per
query, set the six plane distances from the box's bounds and return it.

```text
FUNCTION init_box_hull()
  FOR i in 0..5
    node[i].plane = i
    side = i AND 1                       # alternating: even planes face +, odd face -
    node[i].child[side]     = empty
    node[i].child[side XOR 1] = (i < 5) ? i+1 : solid
    plane[i].axis = i / 2 ;  plane[i].normal[i/2] = 1
FUNCTION hull_for_box(mins, maxs)
  plane distances = maxs.x, mins.x, maxs.y, mins.y, maxs.z, mins.z
  RETURN the hull
```

**Invariants** — **a box is turned into a tree so that one sweep routine serves both.** The source states the reason as
uniformity, and it is the right trade: one tested recursion instead of two. Identical to
[`world.c`](../server/world.c.md)'s.

Note the alternating sides: every plane's normal points along its positive axis, so the maximum planes and the minimum planes
need opposite child assignments. Getting the alternation wrong yields a hull that is inside-out and a player who falls through
box entities only.

## `PM_HullPointContents`, `PM_PointContents`

**Contract** — descend a hull to find what material is at a point; and answer the same for the world, using the world's
walkable hull.

**Invariants** — an axis-aligned plane is tested by **one subtraction rather than a dot product**, selected by the plane's
recorded axis. Same optimization as everywhere in the collision code, and it matters because this is the innermost loop of the
liquid checks that run three times per movement step.

The point query consults **only the world**, not the other solid things in the list. So a player standing inside a moving
platform is still reported as in open air — which is correct for the liquid tests this is used for, and would be wrong if it
were used for anything else.

## `PM_RecursiveHullCheck`

**Contract** — sweeps a point through a hull from one position to another, returning how far it got, the plane it hit and
whether it began or remained inside solid matter.

**Invariants** — the same recursion as [`world.c`](../server/world.c.md): split the segment at the plane, recurse on the near
half, and if that half completes, recurse on the far half; on finding solid matter, back the endpoint off along the plane by a
small epsilon and, if that puts it behind the start, keep backing off. Read that twin for the reasoning — in particular for
why the back-off loop must handle overshooting past the start.

**The epsilon must be identical on client and server.** It is the distance at which the player is considered to be resting
against a surface, and a different value gives a different final position, which gives a prediction that never quite agrees.
That is the general rule for every constant in this file.

## `PM_PlayerMove`

**Contract** — sweeps the player's box from one point to another against every solid thing in the world list, returning the
nearest impact with the tag of what was hit.

```text
FUNCTION player_move(start, end) -> trace
  best = a trace that reached the end
  FOR EACH entry in the world list
    IF the entry is a brush model   hull = its walkable precompiled hull
                                    offset = its origin
    ELSE                            hull = hull_for_box(its bounds EXPANDED by
                                              the player's own bounds)
                                    offset = its origin
    sweep the POINT (start - offset) to (end - offset) through that hull
    IF it started solid or went less far than the best so far
      keep it, and record this entry's tag
  RETURN the best
```

**Invariants** —

- **A box sweep becomes a point sweep by expanding the obstacle by the mover's size** — the Minkowski sum, the same reduction
  as [`world.c`](../server/world.c.md). For the world itself no expansion is needed, because the map compiler already
  precompiled a hull at the player's exact size ([`model.c`](../server/model.c.md)). That precompiled hull is why this is
  cheap and why the player's box size is frozen into the map format.
- **The list is scanned linearly and the nearest impact wins.** At most 32 entries ([`pmove.h`](pmove.h.md)), no spatial
  structure, which is affordable precisely because the caller pre-filtered. The server's own collision code needs a tree
  because it sweeps everything against everything; the player mover does not.
- **Entry zero is the world by convention** and is the only entry with a precompiled hull.
- The tag of the nearest hit is returned, and the mover passes it back to its caller
  ([`pmove.h`](pmove.h.md)) — which is how the server learns which entity to run a touch handler on without the mover knowing
  what an entity is.

## `PM_TestPlayerPosition`

**Contract** — reports whether the player's box, placed at a point, is free of every solid thing in the list.

**Invariants** — implemented as a point-in-hull test against each expanded obstacle, not as a zero-length sweep, because a
sweep of zero length has no direction to back off along. Used by the unsticking search in
[`pmove.c`](pmove.c.md#nudgeposition), where the *order* in which positions are tried matters as much as the test itself.

**Notes** — the whole file is a re-expression of [`world.c`](../server/world.c.md) against a different world representation,
and the duplication is real: two implementations of one collision model that must agree exactly. A rebuild should write the
collision model **once**, over an interface like [`pmove.h`](pmove.h.md)'s flat list, and have the server fill that list from
its tree. That removes the whole class of divergence bug this duplication invites, and it is the clearest improvement a
rebuilder can make over the original.
