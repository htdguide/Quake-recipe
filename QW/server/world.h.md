# QW/server/world.h

> The collision interface: the trace result, the solid classifications, and the operations the server and game logic use to ask about the world.

**Needs** — nothing
**Used by** — [`world.c`](world.c.md) · [`sv_phys.c`](sv_phys.c.md) · [`sv_user.c`](sv_user.c.md) · [`pr_cmds.c`](pr_cmds.c.md)
**Tier floor** — none

## Purpose

Read [`world.h`](../../WinQuake/world.h.md) for the interface and the solid classifications.

## State

As [`world.h`](../../WinQuake/world.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**The trace operation takes the entity to ignore explicitly**, and there is an additional query for the contents at a point that
skips a named entity — both supporting the projectile-versus-owner case ([`world.c`](world.c.md)).

**A trace records which entity's surface was hit as well as the entity itself**, used by the game logic to identify a surface's
material.

**Notes** — a small delta. The interface's shape — a sweep returning a fraction, a plane and an entity — is the durable content
and it is the same in both.
