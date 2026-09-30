# qw-qc — the game logic

> Ten thousand lines in a language with arithmetic, comparison and one timer per entity, defining what the game's rules actually are.

The engine chapters say how a world is simulated and transmitted. This one says what the world *is*: what damage does, what a pickup
gives, when a level ends, how a door decides to crush you. It is written in a small compiled language and runs on the interpreter in
[`../QW/server/pr_exec.c`](../QW/server/pr_exec.c.md).

These are the sources that build `qwprogs.dat`; the build directory is
[`../QW/progs/`](../QW/progs/README.md), which points here.

**Why it is a separate chapter.** A rebuilder replacing the engine keeps this unchanged. A rebuilder making a different game replaces
this and keeps the engine. The boundary between them is [`defs.qc`](defs.qc.md), and it is sharp.

## What the language gives you, and what it does not

Read [`defs.qc`](defs.qc.md) first. The constraints below shape every file in the chapter and explain why the code looks as it does.

- **No records, so a function that must return several values writes globals.** The collision trace returns nine globals, valid only
  until the next trace.
- **No transcendental functions**, so square root and trigonometry are engine operations.
- **One timer per entity** — a single "think at this time" field. Therefore **animation and behaviour are the same mechanism**: every
  sequence is a chain of functions, one per frame, each naming its successor
  ([`player.qc`](player.qc.md)), and every delayed action becomes a spawned entity
  ([`subs.qc`](subs.qc.md)).
- **Four handler fields per entity** — touch, use, think, blocked — and that is the entire callback surface.
- **Ten lifecycle callbacks** the engine calls, listed in [`defs.qc`](defs.qc.md). There is no per-entity frame callback and no
  end-of-level callback.
- **Sixteen numbers of persistence** across a level change, hand-packed
  ([`client.qc`](client.qc.md)).
- **Everything must be precached during level load** ([`world.qc`](world.qc.md)), because asset indices are sent as part of the session's
  description and travel as bytes.
- **The operation table is numbered and append-only forever**, because the numbers are compiled into this code.

## Read in this order

This is also the compilation order ([`progs.src`](progs.src.md)), and it is a dependency order.

**1. [`defs.qc`](defs.qc.md) — the contract.** The globals the engine sets, the callbacks it calls, every shared entity field, and every
operation the game may request. Read it beside
[`../QW/server/progdefs.h`](../QW/server/progdefs.h.md), which is generated from it: the same declarations from the engine's side.

**2. [`subs.qc`](subs.qc.md) — the shared machinery.** Four ideas that the whole map-entity system is built from: *move by velocity and
snap on arrival* (the snap is what stops a door drifting until it no longer closes), *keep a private clock so a blocked mover loses no
time*, *bind targets by name at fire time*, and *schedule by spawning an entity*.

**3. [`combat.qc`](combat.qc.md) — the rules of harm.** Armour as a fraction plus a capacity; knock-back proportional to damage taken,
from the inflictor's *centre*, which is what makes rocket jumping work; a linear blast falloff because physical correctness would play
worse; reachability tested at the target's corners as well as its centre, which is what makes cover behave as it looks.

**4. [`items.qc`](items.qc.md) — pickups.** *An item refuses to be taken when it would be wasted*, which makes items resources you leave
for later. *Hide rather than remove*, which preserves an entity's network identity. And one flag switches the game between contested
renewable resources and depletable ones.

**5. [`weapons.qc`](weapons.qc.md) — the weapons.** Two firing models, instant and projectile. The **damage accumulator** that makes a
shotgun's pellets one damage event per target — without it a target is knocked across the room once per pellet. And the coupling by
which **an animation's length is the rate of fire**.

**6. [`world.qc`](world.qc.md) — the level and the frame hook.** The precache list every map must declare, the light styles, and the
queue of recent corpses.

**7. [`client.qc`](client.qc.md) — the player's lifecycle.** The **pre-think/post-think ordering rule**: whatever must influence movement
goes before, whatever reacts to it goes after. Spawn point selection, the sixteen-number persistence, the level's end conditions, the
intermission.

**8. [`spectate.qc`](spectate.qc.md) — the spectator.** Four functions, and their brevity is the design.

**9. [`player.qc`](player.qc.md) — animations.** The idiom, and the observation that every piece of game logic is written inside an
animation frame because an entity has one timer.

**10. The map entities** — a state machine over the shared mover, with requirements as spawn flags and effects as a target list.
[`doors.qc`](doors.qc.md) (the template; **geometric auto-grouping at load** is the convenience that makes the map format usable) →
[`buttons.qc`](buttons.qc.md) (the minimal example) →
[`plats.qc`](plats.qc.md) (lifts and waypoint chains) →
[`triggers.qc`](triggers.qc.md) (**where a level's logic actually lives**: re-arm intervals rather than per-frame effects, late-bound
destinations, and a relay to work around a single target field) →
[`misc.qc`](misc.qc.md) (the long tail, and the one chain worth tracing end to end: a switchable light, from a game-logic string change
through the renderer's surface invalidation).

**11. [`server.qc`](server.qc.md) — the stubs.** Every monster class, each removing itself. **Read this to calibrate the chapter:
QuakeWorld is the original game with single player removed.**

## Data and generated files

[`progs.src`](progs.src.md) — the build list, which is the reading order above. Two files in the directory are *not* in it.
[`progdefs.h`](progdefs.h.md) — generated from [`defs.qc`](defs.qc.md); never edit it.
[`files.dat`](files.dat.md) — generated: the asset inventory, derived from the precache calls rather than maintained beside them.
[`models.qc`](models.qc.md) · [`sprites.qc`](sprites.qc.md) — read by the art tools, not by the game. The first carries a real hazard:
**a model's frame ordering is an unwritten contract between the art pipeline and the game logic.**

## Cycles

- **[`combat.qc`](combat.qc.md) ↔ [`items.qc`](items.qc.md) ↔ [`weapons.qc`](weapons.qc.md).** Damage reads the attacker's powerups,
  items grant weapons, weapons deal damage. The language resolves it with forward declarations in
  [`defs.qc`](defs.qc.md), which is where the cycle is broken for reading too.
- **[`client.qc`](client.qc.md) ↔ [`player.qc`](player.qc.md).** The lifecycle chooses animations and the animations end in lifecycle
  transitions. Broken at the animation idiom.

## Not twinned

`qwprogs.dat`, the compiled output. Its format is described by
[`../QW/server/pr_comp.h`](../QW/server/pr_comp.h.md) and [`../QW/server/progs.h`](../QW/server/progs.h.md).
