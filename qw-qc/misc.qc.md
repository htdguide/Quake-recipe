# qw-qc/misc.qc

> The remaining map entities: lights, ambient sounds, particle fountains, breakables, the wall that becomes a door, explosions, and the level's decorations.

**Needs** — [`defs.qc`](defs.qc.md) · [`subs.qc`](subs.qc.md) · [`combat.qc`](combat.qc.md)
**Used by** — the map's entity list
**Tier floor** — none

## Purpose

The long tail of the map format. Each class is a few lines and the file's value is as an inventory of what a level designer can place.
Read [`subs.qc`](subs.qc.md) and [`triggers.qc`](triggers.qc.md) first; most of these are trigger or mover variants.

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## The lights

**Contract** — a light entity that the map compiler consumes and the game removes at spawn, unless it has a light style — in which case
it stays and its style can be switched by a trigger.

**Invariants** — **most lights are removed at level start.** Lighting is baked into the map at compile time
([`model.c`](../QW/server/model.c.md)), so a light entity has no run-time meaning unless it is switchable. Keeping them would consume
entity slots for nothing, and the slot limit is real
([`bothdefs.h`](../QW/client/bothdefs.h.md)).

**A switchable light is a style number and a trigger that changes the style's string**
([`world.qc`](world.qc.md) declares the styles, [`r_light.c`](../QW/client/r_light.c.md) evaluates them, and
[`gl_rsurf.c`](../QW/client/gl_rsurf.c.md) or [`r_surf.c`](../QW/client/r_surf.c.md) invalidates the affected surfaces). **That chain —
game logic changes a string, the renderer notices a style level changed, the lit surface is rebuilt — is the whole of dynamic lighting
from static lightmaps**, and it is worth tracing once.

## The ambient sounds

**Contract** — a looping sound attached to a position, started at level load and never stopped.

**Invariants** — **a static sound is sent once in the level description and then owned by the client**
([`sv_init.c`](../QW/server/sv_init.c.md), [`snd_dma.c`](../QW/client/snd_dma.c.md)), costing nothing per frame. It is the cheapest
thing in the game and the reason a level can be full of ambience.

## The breakables and the moving walls

**Contract** — a brush with health that spawns debris when destroyed; and a wall that can be made to appear or disappear, or that moves
when triggered.

**Invariants** — **a wall that appears must be made solid *and* visible in the same operation**, and the engine must be told so it
relinks it into the collision world ([`defs.qc`](defs.qc.md)'s engine-maintained fields). Setting the model without relinking leaves a
visible wall a player walks through — a classic modification bug.

## The explosions and effects

**Contract** — an explosion at a point, with its sound, its radius damage and its visual message; and the various one-off effects a map
can trigger.

**Invariants** — **the visual is a temporary-entity message routed to whoever can see the point**
([`sv_send.c`](../QW/server/sv_send.c.md)) and the damage is computed server-side
([`combat.qc`](combat.qc.md)). The two are independent, which is why a player can be hurt by an explosion they did not see.

## The decorations and the remaining classes

**Contract** — bubbles, teleport effects, the various flame and torch models, and the entities that exist only to be looked at.

**Invariants** — **a purely visual entity still costs an entity slot and a place in every nearby client's snapshot**
([`sv_ents.c`](../QW/server/sv_ents.c.md)). A room full of torches is bandwidth. That coupling between decoration and network cost is
worth stating, because it is invisible to a level designer and it is the reason the snapshot has a per-packet entity cap.

**Notes** — the file is an inventory rather than an algorithm. Its one genuinely load-bearing chain is the switchable light, traced above:
it is the only place in the whole recipe where game logic reaches into the renderer's caching, and following it end to end explains both
the light-style mechanism and the surface invalidation.
