# qw-qc/progs.src

> Data: the game logic's build list — the output file and the source files in compilation order.

**Needs** — nothing
**Used by** — the game-logic compiler
**Tier floor** — none

## Purpose

The build, and therefore the authoritative reading order for this chapter.

## State

Data; no run-time state.

## What it records

```text
the output file name
then, in order:
  defs.qc       -- the interface; MUST be first
  subs.qc       -- the shared mover and target machinery
  combat.qc     -- damage
  items.qc      -- pickups
  weapons.qc    -- weapons
  world.qc      -- the level entity and the frame hook
  client.qc     -- the player lifecycle
  spectate.qc   -- the spectator
  player.qc     -- animations
  doors.qc  buttons.qc  triggers.qc  plats.qc  misc.qc   -- the map entities
  server.qc     -- the stubs, LAST
```

**Invariants** —

- **The order is the compilation order and it matters.** The language has no separate declaration and definition, so a function must be
  declared before it is called — either by appearing earlier, or by a forward declaration in
  [`defs.qc`](defs.qc.md). Reordering the list breaks the build.
- **The interface file must be first**, because the compiler emits the engine's shared globals and fields from it and their *offsets* are
  determined by their order of appearance ([`progdefs.h`](progdefs.h.md)).
- **Two files in the directory are absent from the list**: [`models.qc`](models.qc.md) and
  [`sprites.qc`](sprites.qc.md), which are read by the art tools instead. A reader will otherwise assume they are part of the game.
- The stub file is last, which is arbitrary.

**Notes** — use this list as the chapter's reading order. It is also the dependency order, which is what the recipe asks for.
