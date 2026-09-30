# QW/client/cl_tent.c

> Short-lived visual effects with no entity behind them: explosions, lightning beams and the sparks and trails the server asks for by kind.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`protocol.h`](protocol.h.md) · [`r_part.c`](r_part.c.md) · [`sound.h`](sound.h.md)
**Used by** — [`cl_parse.c`](cl_parse.c.md) on a temporary-entity message; [`cl_main.c`](cl_main.c.md) advances them each frame
**Tier floor** — none

## Purpose

Read [`cl_tent.c`](../../WinQuake/cl_tent.c.md) for the substance: a temporary entity is a kind plus a position, expanded by the
client into a sound, a particle burst and possibly an animated model, and the beams are re-aimed each frame at their owner.

## State

As [`cl_tent.c`](../../WinQuake/cl_tent.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**Explosions are a pool with their own records**, rather than borrowing from the visible entity list. So an explosion outlives the
frame it was spawned in and animates on its own, which matters because the client's frame rate is no longer the server's.

**Beams are keyed by owner and re-aimed from the owner's predicted position**, using the extrapolated player positions
([`cl_ents.c`](cl_ents.c.md)) rather than the last received ones. A lightning beam attached to a player must follow where that player
is *drawn*, not where the last packet put them, or it visibly trails behind them.

**The effect list is cleared explicitly on a level change**, because a level change is no longer a disconnection
([`cl_main.c`](cl_main.c.md)) and stale beams would persist into the new map.

**Invariants** — **an effect's lifetime is in client time, not server time**, so effects are smooth at any frame rate and are
unaffected by packet loss. That is the general rule for anything purely visual: **spawn it from the server, run it on the client.**

**Notes** — the temporary-entity kinds are a numbered protocol contract
([`protocol.h`](protocol.h.md)), and the client's expansion of each kind into sounds and particles is the client's own choice — which
is why two clients can look different while playing the same game.
