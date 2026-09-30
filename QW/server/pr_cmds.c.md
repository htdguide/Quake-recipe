# QW/server/pr_cmds.c

> The game logic's interface to the engine, extended for a networked server: message routing by audience, configuration dictionary access, a kill log, trigonometry, and a projectile trajectory query.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`progs.h`](progs.h.md) · [`pr_comp.h`](pr_comp.h.md) · [`server.h`](server.h.md) · [`world.h`](world.h.md) · [`sv_send.c`](sv_send.c.md) · [`sv_nchan.c`](sv_nchan.c.md)
**Used by** — [`pr_exec.c`](pr_exec.c.md) dispatches every engine call in the game logic to this table
**Tier floor** — none

## Purpose

Read [`pr_cmds.c`](../../WinQuake/pr_cmds.c.md) for the interface's shape and for the eighty-odd operations the game logic can
call — vector maths, entity creation and removal, tracing, spawning, sound, printing, the field iteration, the string handling and
the precaching. Most are unchanged. This twin records the additions and the two removals, because they define what a *networked*
game modification can do that a single-player one cannot.

## State

As [`pr_cmds.c`](../../WinQuake/pr_cmds.c.md); the records are unchanged except where **What differs** says otherwise.

## The message-routing additions

## `PF_multicast`, `PF_Write_*`

**Contract** — the game logic composes a message a byte at a time into a shared buffer, then routes it with a mode: to everyone, to
everyone who can see a point, to everyone who can hear it, or any of those made reliable.

```text
the game logic writes:   WriteByte, WriteChar, WriteShort, WriteLong,
                         WriteAngle, WriteCoord, WriteString, WriteEntity
then calls:              multicast(origin, mode)
```

**Invariants** —

- **The audience is chosen by the game logic, per message.** In the original, a game-logic message goes to one client or to all
  ([`pr_cmds.c`](../../WinQuake/pr_cmds.c.md)); here it can be restricted to those who can see or hear a position, using the
  precomputed sets ([`sv_init.c`](sv_init.c.md)). That is how a modification with many players avoids sending every effect to all
  of them, and **it is a bandwidth decision delegated to the game**, which is unusual and correct: only the game knows whether an
  effect matters at a distance.
- The write operations target **a shared composition buffer**, and the routing call is what copies it into each recipient's stream
  ([`sv_send.c`](sv_send.c.md)). So a write with no matching route leaks into the next message — the write/route pair must be
  used together, and a modification that forgets to route corrupts the stream. That fragility is inherent to a byte-at-a-time
  interface, and a rebuild should scope the buffer to the routing call.
- The destination selector in the original — which client, or broadcast — is retained for compatibility, so old game logic still
  works.

## `PF_infokey`

**Contract** — reads a named entry from a player's information dictionary, or from the server's, returning text.

**Invariants** — **this is how a modification reads a player's name, colours, chosen settings and its own custom keys.** Together
with the server's local dictionary ([`sv_ccmds.c`](sv_ccmds.c.md)) it is the entire configuration channel between the operator and
the game logic — which is why the dictionaries exist rather than a fixed set of settings.

The distinction between the public and local dictionaries is preserved here: the game logic can read both, and the operator
therefore has a private place to put a value the game needs and players must not see.

## `PF_logfrag`

**Contract** — appends a kill record — who killed whom — to the kill log buffer.

**Invariants** — the log is an **external contract with score-tracking software** that the engine knows nothing about
([`sv_ccmds.c`](sv_ccmds.c.md)), and the game logic is what decides what counts as a kill. Recorded because nothing else documents
the format, and because a rebuild will otherwise not think to provide it.

## `PF_setspawnparms`

**Contract** — loads a named player's persistent parameters into the game logic's globals.

**Invariants** — the parameters carry across level changes ([`sv_init.c`](sv_init.c.md)), and in a server that never restarts they
are the only per-player persistence there is.

## `PF_TraceToss`

**Contract** — simulates a tossed entity's whole trajectory under gravity until it stops or hits something, and reports where.

**Invariants** — it runs the **actual toss physics** ([`sv_phys.c`](sv_phys.c.md)) on a copy rather than solving a parabola, so the
answer matches what will really happen including bounces. That is why it is an engine operation and not game logic: the game logic
cannot reproduce the physics exactly.

## `PF_WaterMove`, `PF_changepitch`, `PF_etos`, `PF_sin`, `PF_cos`, `PF_sqrt`, `PF_stof`

**Contract** — run the standard drowning and liquid-damage handling; turn an entity toward a target pitch at its own turn rate;
render an entity reference as text; and the four numeric operations the language lacks.

**Invariants** — **trigonometry and square root are engine operations because the language has no way to compute them.** The game
language ([`pr_comp.h`](pr_comp.h.md)) has arithmetic and comparison and nothing else, so every transcendental function must be an
engine call. That is a property of the language a rebuild inherits if it keeps the language, and a reason to reconsider if it does
not.

The drowning handling being an *engine* operation rather than game logic is a compatibility decision: it was game logic in the
original and was moved so that every modification behaved the same way in water.

## What was removed

**The single-player and saved-game operations are gone**: changing level with the game's own sequencing, and anything touching a
saved game. A public server has no saved games and its level changes are the operator's
([`sv_ccmds.c`](sv_ccmds.c.md)).

**Centre-printing to a specific client and the single-player-only helpers** are reduced to their networked forms.

## `PF_Find`, `PF_VarString`, `PF_Fixme`, `PF_particle`

**Contract** — the field search, the variadic string assembly used by every printing operation, the handler for an unimplemented
slot, and the particle-effect message.

**Invariants** — **an unimplemented operation is a runtime error naming its number**, not a crash, so game logic built against a
newer engine fails informatively on an older one. The builtin table is a **numbered contract**: a number's meaning can never
change, only be added to. That numbering is part of the compiled game logic and is the single hardest thing to change about this
interface.

**Notes** — the table is the engine's entire public interface to the game, and the additions above say precisely what QuakeWorld
decided a networked game needs: **choose your own audience, read configuration, log results, and ask the engine to simulate.** A
rebuild designing a game-logic interface from scratch would do well to start from that list.
