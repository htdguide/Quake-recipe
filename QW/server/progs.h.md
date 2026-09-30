# QW/server/progs.h

> The compiled game logic's in-memory shape: the program header, the definition tables, the entity record with the engine's own fields in front of the game's.

**Needs** — [`pr_comp.h`](pr_comp.h.md) · [`progdefs.h`](progdefs.h.md)
**Used by** — [`pr_exec.c`](pr_exec.c.md) · [`pr_edict.c`](pr_edict.c.md) · [`pr_cmds.c`](pr_cmds.c.md) · [`server.h`](server.h.md)
**Tier floor** — T1: an entity is a variable-sized record addressed by byte offset, and the game's fields are laid out immediately after the engine's

## Purpose

Read [`progs.h`](../../WinQuake/progs.h.md) for the layout and for why an entity reference is a byte offset rather than an index.

## State

As [`progs.h`](../../WinQuake/progs.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**The engine's own per-entity fields gained the networking ones**: which leaves the entity touches and how many, the frame it last
ran physics in, and the owner used to exclude a projectile from its own firer. The leaf list in particular is now part of the
networking ([`sv_ents.c`](sv_ents.c.md)), not only of collision.

**The free-and-reuse timing field** is retained and its role is unchanged.

**Notes** — the entity record's layout — engine fields, then the game's fields as the compiled program declared them — is the
binary contract [`pr_edict.c`](pr_edict.c.md) verifies with a checksum. Adding an engine field changes every offset, which is why
the engine fields are in front and why the two engines' game logic is not interchangeable.
