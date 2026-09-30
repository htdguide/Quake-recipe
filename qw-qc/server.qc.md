# qw-qc/server.qc

> Placeholders for every entity class the maps reference but this game does not implement: the monsters, and the path and event entities.

**Needs** — [`defs.qc`](defs.qc.md)
**Used by** — the map's entity list
**Tier floor** — none

## Purpose

The single most informative file in the chapter about what QuakeWorld *is*. It declares every monster class the original game's maps
contain — and each one is an empty function that removes itself.

**QuakeWorld has no monsters.** The maps still name them, so the classes must exist or the map load fails
([`pr_edict.c`](../QW/server/pr_edict.c.md) reports an unknown class), and each one deletes itself at spawn.

## State

Stateless — every class here removes itself at spawn.

## The classes

**Contract** — every monster class, plus the lightning event and the waypoint and movement-target classes, each removing itself.

**Invariants** —

- **A class named by a map must exist even if it does nothing.** That is a map-format contract: the entity list is text naming classes
  ([`pr_edict.c`](../QW/server/pr_edict.c.md)), and an unknown name is reported at every level load. So deleting a feature means
  replacing its classes with stubs, not removing them.
- **The monster artificial intelligence is absent from the whole chapter.** The engine still carries the monster movement helpers
  ([`sv_move.c`](../QW/server/sv_move.c.md)) and the interpreter still exposes them
  ([`pr_cmds.c`](../QW/server/pr_cmds.c.md)) — dead code in a deathmatch-only game, kept so that a modification can add creatures back.
- **The path and movement-target classes are stubs here** but the train's waypoints are not
  ([`plats.qc`](plats.qc.md)) — the two mechanisms are separate and only the monster one is gone.

**Notes** — read this file to calibrate the whole chapter. **The QuakeWorld game logic is the original game with single player removed:
no monsters, no saved games, no campaign sequencing, no cut scenes — and everything that remains is the competitive game.** That is why
the chapter is ten thousand lines rather than thirty, and why [`items.qc`](items.qc.md)'s respawning and
[`client.qc`](client.qc.md)'s scoring carry the weight that monster behaviour carries in the original.

For a rebuilder the useful consequence: **if you are rebuilding the competitive game, this chapter is the complete specification and you
need nothing of the original's monster logic.** If you want single player, none of it is here.
