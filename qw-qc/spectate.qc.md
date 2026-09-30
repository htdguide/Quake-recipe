# qw-qc/spectate.qc

> The spectator's game logic: almost nothing, which is the point.

**Needs** — [`defs.qc`](defs.qc.md) · [`client.qc`](client.qc.md)
**Used by** — the engine calls its callbacks for a spectator instead of the player ones ([`sv_user.c`](../QW/server/sv_user.c.md))
**Tier floor** — none

## Purpose

The game-logic side of spectating, added by QuakeWorld and with no counterpart in the original. Four functions, and their brevity is the
design: **a spectator is a player the game does not simulate.**

## State

The entity fields declared in [`defs.qc`](defs.qc.md); this file adds no globals of its own beyond the constants named below.

## `SpectatorConnect`, `SpectatorDisconnect`

**Contract** — announce a spectator joining and leaving, and give them a non-solid, non-damageable, invisible state.

**Invariants** — a spectator is **not solid, cannot be damaged, has no model and no weapon**. Everything that makes a player a
participant is switched off here, and the engine additionally skips their movement's world interaction and their visibility to others
([`sv_user.c`](../QW/server/sv_user.c.md), [`sv_ents.c`](../QW/server/sv_ents.c.md)).

## `SpectatorThink`

**Contract** — called instead of the player's two think callbacks. Does nothing but let the frictionless flying movement happen.

**Invariants** — **there is one callback rather than two**, because there is no movement to bracket: nothing needs to influence it and
nothing reacts to it. That asymmetry with the player callbacks ([`defs.qc`](defs.qc.md)) is deliberate and it is a useful signal about
what the two player callbacks are actually for.

## `SpectatorImpulseCommand`

**Contract** — handle the numbered commands a spectator may send: cycle which player to follow, and toggle the camera mode. Anything
else is refused.

**Invariants** — **the command set is a whitelist**, so a spectator cannot fire, change weapons or use the developer commands by
sending a player's impulse number. That check is here in the game logic, and the engine separately refuses a spectator's teleport
command from a non-spectator ([`sv_user.c`](../QW/server/sv_user.c.md)). Two checks in two places for one rule, and the recipe records
that a rebuild should have one.

**Notes** — the whole file is worth reading as the answer to a question a rebuild will face: **what is a viewer who is present but not
playing?** The answer here is a client with a channel, a position and a chosen subject, excluded from every simulation and every other
player's view, with the server cooperating by sending them what their subject can see. Four functions in the game logic and three
special cases in the engine.
