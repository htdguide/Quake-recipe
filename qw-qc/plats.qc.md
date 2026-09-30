# qw-qc/plats.qc

> Platforms and trains: a lift that returns when stepped off, and an entity that moves along a path of waypoints.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md)
**Used by** — the map's entity list
**Tier floor** — none

## Purpose

Two movers whose behaviour is defined by what they carry rather than by a trigger. Read
[`subs.qc`](subs.qc.md) for the move-and-snap machinery and [`doors.qc`](doors.qc.md) for the state-machine pattern.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## The platform

**Contract** — a lift with two positions derived from its own height: it rests at the top, descends when a player steps onto its trigger
volume, waits, and returns. A spawn flag makes it rest at the bottom and rise instead.

**Invariants** —

- **The travel distance is computed from the platform's own size**, not declared, so a level designer sizes the brush and the platform
  knows how far to move. The distance can be overridden by a field.
- **The trigger volume is generated from the platform's bounds, inset slightly and raised**, so a player standing on it is inside it and
  a player beside it is not. The inset is what stops a platform triggering from an adjacent walkway.
- **A platform's blocked handler damages the obstruction heavily**, because a lift cannot reverse without stranding whoever is on it.
- The wait at the bottom is fixed, and a player who steps off before it expires still gets the return — the trigger volume's departure is
  not tracked, only its entry.

## The train

**Contract** — an entity that moves from waypoint to waypoint, each waypoint an entity in the map naming the next, pausing at each for a
declared time, and looping. A spawn flag makes it start stopped until triggered.

```text
FUNCTION train_next()
  target = find the entity named by our current target
  IF none  report the broken path
  our new target = its target                  # follow the chain
  destination = its origin - our bounding box's minimum corner
  wait = its declared wait
  calc_move(destination, our speed, train_next)
```

**Invariants** —

- **A train's path is a chain of named entities, resolved one hop at a time at run time**
  ([`subs.qc`](subs.qc.md)'s name binding). So a path can branch by rewiring a waypoint's target while the train runs, and a broken chain
  is reported rather than crashing.
- **The destination is offset by the train's own bounding box**, because a waypoint marks where the train's *corner* goes, not its
  centre. That convention is a map-format contract and getting the sign wrong displaces every train by its own size.
- **The train's first move happens after a frame's delay**, so every waypoint has spawned before the chain is followed.
- **The wait comes from the waypoint, not the train**, so the pause can differ at each stop.

**Notes** — the waypoint chain is the same late-bound-by-name idea as the target graph
([`subs.qc`](subs.qc.md)), applied to motion. One mechanism, two uses, and both are what make the map format a wiring language rather
than a set of fixed behaviours.
