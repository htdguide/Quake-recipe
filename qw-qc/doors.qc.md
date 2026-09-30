# qw-qc/doors.qc

> Doors: a two-state mover with a linked group that opens together, a key or trigger requirement, a blocked reaction, and the secret-door variant with its sequenced motion.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md) · [`combat.qc`](combat.qc.md)
**Used by** — the map's entity list
**Tier floor** — none

## Purpose

The richest of the map-entity files and the template for every other mover. Read
[`subs.qc`](subs.qc.md) first: a door is its move-and-snap machinery plus a state machine plus a grouping rule.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## The state machine

```text
states:  bottom -> going up -> top -> going down -> bottom
a trigger or a touch while at bottom  starts going up
arriving at the top   schedules going down after the wait, or stays if the wait is -1
arriving at the bottom  is the resting state
a trigger while going down  reverses to going up
```

**Invariants** —

- **Four states, and the transitions are asymmetric**: a door part-way down reverses, a door part-way up does not. Without the reversal
  a player standing in a closing door is crushed by a door they just triggered.
- **A wait of a distinguished value means "stay open forever"**, which is how a door that must be opened once is expressed with the same
  entity.
- The motion itself is the shared move-and-snap ([`subs.qc`](subs.qc.md)), so the door's own code never touches the origin between the
  endpoints.

## The linked group

**Contract** — at level start, every door touching another door with a compatible orientation is linked into a ring, and triggering any
of them triggers all of them.

```text
FUNCTION link_doors()
  IF already linked  RETURN
  start a ring with self
  grow a shared bounding box
  LOOP: find a door whose bounds touch the box
    add it to the ring ;  grow the box ;  mark it linked
  give every door in the ring the group's combined bounds and shared health
```

**Invariants** —

- **Grouping is discovered geometrically at level start, not declared.** A level designer places two door halves next to each other and
  they open together with no wiring. That is the single most convenient thing about the format and it is twenty lines.
- **The group shares one trigger volume and one health total**, so shooting either half opens both and a trigger field covers the pair.
- **A door with a trigger volume generates that volume from the group's bounds**, expanded, so the player need not touch the door itself.
- The linking must run **before any door can be triggered**, which is why it happens on the first frame rather than at spawn.

## The requirements

**Contract** — a door may require a key, or may open only when triggered rather than touched, or may require being shot.

**Invariants** — **each requirement is a spawn flag or a field on the entity**, read by the touch handler. So one class covers a
touch-opened door, a locked door, a shootable door and a remotely triggered one, and the difference is bits the map author sets. That is
the map format's whole extensibility mechanism ([`defs.qc`](defs.qc.md)'s spawn flags), and every entity file uses it the same way.

A locked door **prints its own message and plays its own sound on refusal**, both authored per entity.

## The blocked reaction

**Contract** — when the engine reports the door could not move because something was in the way
([`sv_phys.c`](../QW/server/sv_phys.c.md)), damage the obstruction; and if the door has no damage set, reverse instead.

**Invariants** — **the blocked handler is the engine's callback and the door's whole crushing behaviour** is these few lines. A door
with damage crushes; one without reverses. And because the engine's push is all-or-nothing
([`sv_phys.c`](../QW/server/sv_phys.c.md)), a blocked door has not moved at all when this is called — which the original engine cannot
guarantee.

## The secret door

**Contract** — a variant that moves sideways and then backwards in a timed sequence, optionally requiring a shot rather than a touch,
and counting toward the level's secret total.

**Invariants** — the sequence is expressed as **a chain of move-and-snap calls, each one's arrival callback starting the next**
([`subs.qc`](subs.qc.md)). That is the idiom for any multi-step motion in this language: there are no coroutines, so a sequence is a
linked list of callbacks. A rebuild with any form of suspension will write this as a script and should notice that the original could
not.

**Notes** — the two ideas worth taking are **geometric auto-grouping at load** and **spawn flags as the extensibility mechanism**. Both
are what let a level designer build behaviour without writing code, which is the map format's purpose.
