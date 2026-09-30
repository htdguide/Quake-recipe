# QW/server/pr_edict.c

> Loading the compiled game logic and managing the entity pool: the field definition tables, entity allocation and freeing, and the text form used for the map's entity list.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`progs.h`](progs.h.md) · [`pr_comp.h`](pr_comp.h.md) · [`progdefs.h`](progdefs.h.md) · [`crc.h`](../client/crc.h.md)
**Used by** — [`sv_init.c`](sv_init.c.md) · [`sv_main.c`](sv_main.c.md) · [`pr_cmds.c`](pr_cmds.c.md)
**Tier floor** — none

## Purpose

Read [`pr_edict.c`](../../WinQuake/pr_edict.c.md) for the whole substance: the compiled program's layout, the field and global
definition tables, the name lookups, the entity pool with its free-and-reuse delay, and the key-value text form the map's entity
list is written in.

## State

As [`pr_edict.c`](../../WinQuake/pr_edict.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**The saved-game reading and writing are gone.** A public server does not save.

**The required-field check is against QuakeWorld's own field set** ([`progdefs.h`](progdefs.h.md)), which is larger, and the
checksum of the field layout is compared against a known value — so game logic compiled for the original engine is refused with a
clear message rather than corrupting memory. That check is the right thing and a rebuild should keep it: **a compiled
program's field layout is a binary contract, and the checksum is how it is enforced.**

**The entity pool's reuse delay is unchanged** but matters more: on a server running for weeks, entities are created and destroyed
continuously, and reusing a slot too soon lets a stale reference in game logic address a different entity.

**Notes** — the definition tables are also what the server uses to expose entity fields to the operator
([`sv_ccmds.c`](sv_ccmds.c.md)), which is the only debugging facility a live server has.
