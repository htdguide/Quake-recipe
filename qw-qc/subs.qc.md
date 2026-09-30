# qw-qc/subs.qc

> The shared machinery every moving and triggering entity uses: a linear move to a destination with a callback on arrival, the target-firing graph, and the delayed trigger.

**Needs** — [`defs.qc`](defs.qc.md)
**Used by** — [`doors.qc`](doors.qc.md) · [`plats.qc`](plats.qc.md) · [`buttons.qc`](buttons.qc.md) · [`triggers.qc`](triggers.qc.md) · [`misc.qc`](misc.qc.md)
**Tier floor** — none

## Purpose

The four ideas the whole map-entity system is built from. Read this before any of the entity files; every door, platform, button and
trigger is these four in combination.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## `SUB_CalcMove`, `SUB_CalcMoveEnt`, `SUB_CalcMoveDone`

**Contract** — move an entity from where it is to a destination at a given speed, calling a named function on arrival. Sets a velocity
and a thinking time; the engine does the moving.

```text
FUNCTION calc_move(destination, speed, on_arrival)
  remember the destination and the callback
  delta = destination - origin
  traveltime = |delta| / speed
  IF traveltime < the frame quantum
    schedule arrival next frame with zero velocity
    RETURN
  nextthink = ltime + traveltime            # this entity's OWN clock
  velocity = delta / traveltime
  think = on_arrival

FUNCTION calc_move_done()
  origin = the remembered destination         # SNAP, do not drift
  velocity = 0
  nextthink = -1
  call the remembered callback
```

**Invariants** —

- **Movement is expressed as a velocity plus an arrival time, not as a per-frame step.** So the engine does the integration and the
  collision ([`sv_phys.c`](../QW/server/sv_phys.c.md)) and the game is only involved at the endpoints. That is what keeps a door's logic
  to a few lines and why the game logic runs a hundred thousand times a frame's worth of work rather than millions
  ([`profile.txt`](../QW/server/profile.txt.md)).
- **The arrival snaps the origin to the exact destination.** Integrating a velocity for a computed time does not land exactly, and
  without the snap a door drifts a fraction further open every cycle until it no longer closes. *This is the single most important line
  in the file.*
- **The timing uses the entity's own clock, not the world's.** A pushing entity has a local clock the engine advances only when it moves
  ([`defs.qc`](defs.qc.md)), so a door blocked part-way does not lose time. Using the world clock instead makes a blocked door jump when
  it is freed.
- A move shorter than one frame is **scheduled rather than performed**, because a velocity for less than a frame cannot be expressed.

## `SUB_CalcAngleMove`, `SUB_CalcAngleMoveDone`

**Contract** — the same for a rotation, using angular velocity.

**Invariants** — the same snap-on-arrival rule applies, and matters more, because an angle that drifts accumulates visibly.

Note that QuakeWorld's engine removed rotating *pushers*
([`sv_phys.c`](../QW/server/sv_phys.c.md)), so a rotating entity here turns without carrying riders.

## `SUB_UseTargets`

**Contract** — fire everything this entity points at: print its message to whoever activated it, play its sound, then find every entity
whose name matches this entity's target and call each one's use handler — after a delay if one is set, by spawning a temporary entity to
do the firing later. An entity that names a kill-target instead removes those entities.

```text
FUNCTION use_targets()
  IF a delay is set
    spawn a temporary entity carrying the target name and the activator
    give it a think time of now + delay ;  RETURN      # it fires later
  IF a message is set AND the activator is a player  print it to them
  play this entity's noise
  IF a kill-target is set
    FOR EACH entity whose name matches it  remove it
  IF a target is set
    FOR EACH entity whose name matches it
      IF it has a use handler  call it with the activator preserved
```

**Invariants** —

- **The entity graph is by name, resolved at fire time, not at load.** An entity's target is a string and the search is a field
  comparison over the whole entity list ([`pr_cmds.c`](../QW/server/pr_cmds.c.md)). So one name may have many matches, a name may match
  nothing, and the graph may be edited by the map author with no compilation step. **That late binding is the whole extensibility of
  the map format** and it is why a level designer can wire arbitrary behaviour without touching the game logic.
- **A delay is implemented by spawning a temporary entity to do the firing.** There is no timer other than an entity's thinking time
  ([`defs.qc`](defs.qc.md)), so a delayed action *must* become an entity. That is the language's only scheduling primitive and this
  function is the idiom for using it.
- **The activator is carried through the delay**, because messages and scoring need to know who caused the chain. Losing it is the
  obvious bug in the delayed path.
- The kill-target is checked **before** the target, so an entity can remove things and then fire others.

## `InitTrigger`

**Contract** — set up any entity that exists to be touched: make it a trigger-solid, give it its model's bounds, and make it invisible.

**Invariants** — **a trigger has a model only so the map compiler can give it a size**, and the model is then discarded. That is the
convention by which a brush in the map becomes an invisible volume, and it is a map-format contract the engine does not state.

## `SUB_Null`, `SUB_Remove`, `DelayThink`

**Contract** — do nothing; remove this entity; and the temporary entity's handler that performs a delayed firing.

**Invariants** — a do-nothing function exists because a handler field must be settable to something harmless, and **an unset handler
field is a fatal error when called** rather than a no-operation. That is a property of the engine's dispatch
([`pr_exec.c`](../QW/server/pr_exec.c.md)).

**Notes** — the four ideas — *move by velocity and snap on arrival, keep a private clock, bind targets by name at fire time, and
schedule by spawning an entity* — are the whole vocabulary of Quake level design. A rebuild that keeps the map format must keep all
four; one that does not should still recognize the first as the general answer to "animate between two states without drift".
