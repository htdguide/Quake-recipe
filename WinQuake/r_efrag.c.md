# WinQuake/r_efrag.c

> Entity fragments: the two-way cross-reference between an entity and every map leaf it touches, rebuilt whenever an entity moves, and the pass that turns visible leaves into a visible-entity list.

**Needs** — [`r_local.h`](r_local.h.md) · [`render.h`](render.h.md) · [`model.h`](model.h.md) · [`client.h`](client.h.md) · [`mathlib.h`](mathlib.h.md)
**Used by** — [`cl_main.c`](cl_main.c.md) adds and removes fragments as entities move; [`r_bsp.c`](r_bsp.c.md) stores them when the walk reaches a visible leaf
**Tier floor** — none

## Purpose

The renderer's visibility walk finds *leaves*. To draw entities it needs to go from a visible leaf to the
entities in it, and this file is that index: a small record per (entity, leaf) pair, threaded into two linked
lists ([`render.h`](render.h.md)).

The second thing here is the **top node**: the first tree node that splits an entity's bounding box. It
decides whether a sub-model can be drawn unclipped or must be clipped against the world tree
([`r_bsp.c`](r_bsp.c.md)).

## State

```text
VARIABLE r_pefragtopnode : MNode         # the split node being discovered
VARIABLE lastlink : pointer              # where to append the next fragment
VARIABLE r_emins, r_emaxs : vec3         # the entity's world-space box
VARIABLE r_addent : Entity
```

**Invariants** — fragments come from a free list in the client's state
([`client.h`](client.h.md)), 640 of them, and exhausting it silently drops fragments.

## `R_AddEfrags`

**Contract** — takes an entity; walks the world tree inserting a fragment into every leaf its bounding box
touches, and records the first node that splits the box as the entity's top node. Skips an entity with no
model, and never adds the world.

```text
FUNCTION r_add_efrags(ent)
  IF ent has no model  RETURN
  IF ent IS the world  RETURN
  r_addent = ent ;  lastlink = &ent.efrag ;  r_pefragtopnode = nothing
  r_emins = ent.origin + its model's mins
  r_emaxs = ent.origin + its model's maxs
  r_split_entity_on_node(the world's root)
  ent.topnode = r_pefragtopnode
```

## `R_SplitEntityOnNode`

**Contract** — recursion: at a leaf, allocates a fragment and links it into both lists; at an interior node,
tests the box against the plane, remembers the node if this is the first split, and descends into whichever
sides the box reaches.

```text
FUNCTION r_split_entity_on_node(node)
  IF node IS a leaf
    IF it is solid  RETURN
    take a fragment from the free list ;  IF none, RETURN
    link it into the ENTITY's list AT lastlink, and advance lastlink
    link it into the LEAF's list at the front
    RETURN
  sides = box_on_plane_side(r_emins, r_emaxs, node.plane)
  IF sides == 3                                  # the box straddles
    IF r_pefragtopnode IS nothing  r_pefragtopnode = node    # the FIRST splitter
  IF sides HAS the front bit  recurse INTO children[0]
  IF sides HAS the back bit   recurse INTO children[1]
```

**Invariants** — **the entity's list is built in traversal order** by appending through a moving tail pointer,
while each leaf's list is built by prepending. The order of the entity's list is what the drawing pass walks,
and it matters only for determinism.

**The top node is the *first* node whose plane splits the box**, recorded during the descent. An entity
entirely inside one leaf has none, which is the signal that it can be drawn unclipped.

A solid leaf contributes no fragment, so an entity partly inside a wall is only present in the leaves it
actually occupies.

## `R_RemoveEfrags`

**Contract** — takes an entity; unlinks every one of its fragments from the leaf lists that hold them and
returns them all to the free list.

```text
FUNCTION r_remove_efrags(ent)
  FOR EACH fragment ef IN ent's list
    walk ef.leaf's list and splice ef out of it      # a LINEAR search
    return ef to the free list
  ent.efrag = nothing
```

**Invariants** — the leaf's list is singly linked, so removing a fragment requires walking it. With many
entities in one leaf that is quadratic in the leaf's occupancy. It is bounded in practice by the entity count.

**Called on every entity that moves, every frame** ([`cl_main.c`](cl_main.c.md#cl_relinkentities)), followed
immediately by an add — so the whole index is rebuilt each frame for every moving entity. A rebuild that
detects "the box did not change leaves" avoids most of it.

## `R_SplitEntityOnNode2`

**Contract** — a variant used for sub-models: descends only into the side the box is on, stopping at the first
node that splits it or at the first non-solid visible leaf. Ignores nodes not marked visible this frame.

**Invariants** — this is the **visibility-aware** version, and its purpose is different from its sibling's: it
answers "is this sub-model visible, and if so is it clipped" rather than building an index. Descending only
one side is correct because it stops the moment the box straddles.

## `R_StoreEfrags`

**Contract** — takes a leaf's fragment list; for each fragment whose entity has not already been added this
frame, appends that entity to the renderer's visible list and marks it. Silently stops when the list is full.

```text
FUNCTION r_store_efrags(fragments)
  FOR EACH fragment
    ent = its entity
    SELECT its model's kind
      alias, brush, sprite:
        IF ent.visframe != r_framecount AND the visible list is not full
          append ent TO the visible list
          ent.visframe = r_framecount
```

**Invariants** — **the frame stamp is what deduplicates.** An entity spanning four visible leaves is reached
four times and added once. That is the whole reason the entity carries a visibility stamp
([`render.h`](render.h.md)).

The visible list caps at 256 ([`client.h`](client.h.md)) and overflow drops entities silently, favouring those
in leaves the walk reached first — which is to say, nearer ones, because the walk is front to back.

**Notes** — the source's comment says a lot of this goes away with an edge-based renderer, which is what this
engine is. The remark is about the fragment machinery being a holdover from a design where entities were
inserted into the tree rather than drawn in a separate pass.
