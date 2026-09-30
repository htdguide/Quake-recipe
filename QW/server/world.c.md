# QW/server/world.c

> The collision world: entities indexed in a hand-built spatial tree, box sweeps reduced to point sweeps against precompiled or fabricated hulls.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`world.h`](world.h.md) · [`model.c`](model.c.md)
**Used by** — [`sv_phys.c`](sv_phys.c.md) · [`sv_user.c`](sv_user.c.md) · [`sv_move.c`](sv_move.c.md) · [`pr_cmds.c`](pr_cmds.c.md)
**Tier floor** — none

## Purpose

Read [`world.c`](../../WinQuake/world.c.md) in full — the area tree, the Minkowski-sum reduction, the fabricated box hull, the
recursive sweep with its back-off epsilon, the leaf-list recording and the trigger touching are all unchanged and are among the
most important algorithms in the recipe.

## State

As [`world.c`](../../WinQuake/world.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**A trace can be told to ignore an entity's owner**, so a projectile does not collide with whoever fired it on its first frame.
The original handles this in the game logic; here the engine's trace takes the exclusion, which is both faster and less
error-prone.

**The leaf list recorded per entity is used for two purposes now**, not one: the original uses it only for the client's rendering
hand-off, while here it is also what decides whether an entity is in a client's visible or hearable set
([`sv_ents.c`](sv_ents.c.md), [`sv_send.c`](sv_send.c.md)). So the leaf list has become part of the networking, and the limit on
how many leaves are recorded per entity is now a limit on how reliably a large entity is seen. An entity touching more leaves than
the limit is visible only from the leaves that were recorded.

**The area tree's depth and the trigger-touch recursion are unchanged**, and remain the server's second-largest measured cost
([`profile.txt`](profile.txt.md)).

**Notes** — the important observation for a rebuilder is the second one: in QuakeWorld the collision world's leaf bookkeeping does
double duty as the visibility index. Those are two responsibilities in one structure, and they have different requirements — a
collision index wants to be coarse, a visibility index wants to be exact. A rebuild may want to separate them, and should at
least know they are joined.
