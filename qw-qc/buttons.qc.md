# qw-qc/buttons.qc

> Buttons: a mover that fires its targets on arrival, triggered by a touch or a shot, with an optional automatic return.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md)
**Used by** — the map's entity list
**Tier floor** — none

## Purpose

The simplest of the movers, and therefore the clearest demonstration of the pattern. Read
[`subs.qc`](subs.qc.md) and [`doors.qc`](doors.qc.md) first.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## The behaviour

```text
states:  out -> going in -> in -> going back -> out
a touch or a shot while out  starts going in
arriving in  FIRES THE TARGETS, then schedules going back after the wait
a wait of the distinguished value means it stays in forever
```

**Invariants** —

- **The targets are fired on arrival, not on activation.** So a button with a slow travel introduces a delay the level designer can feel,
  and the door it opens begins moving when the button is fully pressed. Firing on activation instead would make travel time cosmetic.
- **The travel distance comes from the button's own size** along its declared direction, like a platform's
  ([`plats.qc`](plats.qc.md)).
- **A shot-activated button is expressed by giving it health**, reusing the damage system rather than adding a mechanism
  ([`combat.qc`](combat.qc.md)). Its death handler is its activation.

## The rotating variant

**Contract** — the same behaviour expressed as a rotation rather than a translation, using the shared angular move
([`subs.qc`](subs.qc.md)).

**Invariants** — a rotating button **does not carry riders**, because QuakeWorld's engine dropped rotating pushers
([`sv_phys.c`](../QW/server/sv_phys.c.md)). A level designer must not build a rotating platform.

**Notes** — the file is short and its value is as the minimal example: **a map entity is a state machine over the shared mover, with its
requirements as spawn flags and its effect as a target list.** Every other entity file is this with more cases.
