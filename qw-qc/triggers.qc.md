# qw-qc/triggers.qc

> Trigger volumes: the many things that happen when a player enters a region — firing targets, teleporting, hurting, changing the level, setting a skill, playing a sound, or pushing.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md) · [`combat.qc`](combat.qc.md)
**Used by** — the map's entity list
**Tier floor** — none

## Purpose

Where a level's logic actually lives. Every one of these is an invisible volume with a touch handler
([`subs.qc`](subs.qc.md)'s trigger setup), and the variety is the level designer's vocabulary.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## The general trigger

**Contract** — fire this entity's targets when touched, once or repeatedly, optionally requiring a key, optionally printing a message,
and optionally only for a player rather than any entity.

**Invariants** —

- **A repeating trigger has a re-arm delay**, or it fires every frame a player stands in it. That delay is the difference between a
  working trigger and a flood.
- **A once-only trigger removes itself**, which also removes it from the entity count — worth noting given the limit
  ([`bothdefs.h`](../QW/client/bothdefs.h.md)).
- **What may activate a trigger is a spawn flag**: players only, or monsters too, or anything. A rocket passing through a trigger is a
  real case and the flag is how a designer decides.

## The teleporter

**Contract** — move the toucher to a destination entity's position and angles, set their velocity along the destination's facing, spawn
the effects at both ends, and **kill anything already standing at the destination**.

```text
FUNCTION teleport_touch()
  IF the toucher may not teleport  RETURN
  destination = the entity named by our target
  spawn the departure effect at the toucher's position
  move the toucher to the destination's origin, raised slightly
  angles = the destination's angles ;  fixangle = true
  velocity = forward * a fixed speed          # you arrive moving
  kill anything already at the destination
  spawn the arrival effect
```

**Invariants** —

- **Arriving kills the occupant.** Same decision as spawning ([`client.qc`](client.qc.md)) and for the same reason: the alternatives are
  worse.
- **The arrival sets a velocity rather than leaving the player still**, which is what makes a teleporter feel like being thrown and, more
  practically, moves the player out of the arrival volume so they do not re-trigger.
- **The forced angle travels unreliably and can be lost**
  ([`docs.txt`](../QW/client/docs.txt.md)), which is a known defect: a teleport occasionally leaves a player facing the wrong way.
- **The destination is bound by name at touch time** ([`subs.qc`](subs.qc.md)), so a teleporter's exit can be rewired during play.
- The effects are entities, not messages, so they are visible to everyone who can see either end.

## `trigger_hurt`, `trigger_push`, `trigger_monsterjump`, `trigger_multiple`, `trigger_once`, `trigger_secret`, `trigger_counter`, `trigger_onlyregistered`, `trigger_setskill`, `trigger_changelevel`, `trigger_relay`, `target_secret`

**Contract** — damage whatever is inside at a rate; add a velocity to whatever enters; the same aimed at monsters; the general trigger in
its repeating and once forms; the secret-found counter; a trigger that fires only after being touched a declared number of times; one
that checks whether the game is registered; one that sets the difficulty; the level change
([`client.qc`](client.qc.md)); and a trigger that fires targets when triggered rather than touched.

**Invariants** —

- **A damaging volume re-arms on a short interval** rather than damaging per frame, so its rate is independent of the server's frame
  rate. That is the general rule for anything continuous in a frame-driven system and it is easy to get wrong.
- **A push volume sets the velocity rather than adding to it** in most cases, so its effect is predictable regardless of how the player
  entered.
- **The counter trigger holds its count in a field and prints the remaining number**, which is the whole of a multi-switch puzzle.
- **A relay exists because a trigger's target list is one field**: firing two unrelated chains from one event needs an intermediary. That
  is a limitation of the name-binding design showing through
  ([`subs.qc`](subs.qc.md)).
- The difficulty trigger writes a **server setting** from the game logic, which is a game modifying the engine's configuration — possible
  because settings are a shared namespace ([`cvar.c`](../QW/client/cvar.c.md)).

**Notes** — the whole file is one pattern with a dozen payloads, and the interesting content is in the invariants: **re-arm intervals
rather than per-frame effects**, **late-bound destinations**, and **a relay to work around a single target field**. A rebuild designing a
level-scripting vocabulary should read this as the list of what a designer actually needs.
