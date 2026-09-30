# qw-qc/world.qc

> The level's own entity: the precaching every map needs, the once-per-frame hook, and the queue of recent corpses.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md)
**Used by** — the engine calls the frame hook; the map's own world entity instantiates the spawn function
**Tier floor** — none

## Purpose

What runs once per level and once per frame, and one small piece of state that has to live somewhere.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## `worldspawn`

**Contract** — the map's world entity's spawn function: set the level's name and the key set from the map's own fields, then **precache
every model and sound the game can possibly use**, then set the light styles.

**Invariants** —

- **Everything must be precached here, during level load, and nothing afterwards.** The engine refuses a precache once the level is
  running ([`server.h`](../QW/server/server.h.md)'s load state), because the precache lists are sent to clients as part of the level
  description ([`sv_init.c`](../QW/server/sv_init.c.md)) and a later addition would be an index the client does not have. So this one
  function must name every asset in the game, whether the map uses it or not.
  ***That is the single most restrictive constraint the engine places on game logic***, and it is why the list is long and why a
  modification adding a model must edit this function.
- **The order of the precache calls is the index assignment**, and indices travel as bytes
  ([`bothdefs.h`](../QW/client/bothdefs.h.md)). So reordering this function changes the protocol's meaning, and the nail special case
  ([`sv_ents.c`](../QW/server/sv_ents.c.md)) depends on particular models being precached at all.
- **Light styles are declared here**, each a string of intensity letters
  ([`r_light.c`](../QW/client/r_light.c.md)), and style zero must be the normal one. The strings are the flicker patterns and they are
  content.

## `StartFrame`

**Contract** — called once per server frame before anything moves: reset the per-frame counters and decrement the retouch countdown.

**Invariants** — **it is the only place a periodic rule can live that is not tied to an entity**, because there is no other global
callback ([`defs.qc`](defs.qc.md)). The level's end conditions are *not* checked here but per player
([`client.qc`](client.qc.md)), which is wasteful and is what the source does.

## `InitBodyQue`, `bodyque`, `CopyToBodyQue`

**Contract** — create a small ring of placeholder entities at level start; and on a player's death, copy their body's appearance and
position into the next placeholder, so the corpse persists after the player respawns.

```text
FUNCTION copy_to_body_que(player)
  slot = the next placeholder in the ring
  copy the player's angles, model, frame, colour map, movetype,
    velocity, origin and size into it
  advance the ring
```

**Invariants** —

- **The corpse is a separate entity because the player's own entity is reused on respawn.** Without the copy, respawning would make the
  body vanish, which reads as the death not having happened.
- **The ring is small and fixed**, so only the last few corpses persist. That bounds the entity count — which matters, because the limit
  is 768 and the protocol addresses 512 ([`bothdefs.h`](../QW/client/bothdefs.h.md)).
- The placeholders are **allocated at level start**, not on demand, because allocation during play competes with projectiles for slots.

## `main`

**Contract** — unused.

**Notes** — the precache constraint is the transferable lesson: **if asset indices are assigned at load and sent as part of a session's
description, then every asset must be declared before the session begins.** A rebuild can lift the restriction by sending names instead
of indices, at a cost in bandwidth, or by allowing mid-session additions — and should decide deliberately, because this one function's
awkwardness is the whole visible consequence.
