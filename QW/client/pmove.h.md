# QW/client/pmove.h

> The shared movement interface: one input record, one world snapshot, one result, so that client and server can run identical player physics.

**Needs** — nothing
**Used by** — [`pmove.c`](pmove.c.md) · [`pmovetst.c`](pmovetst.c.md) · [`cl_pred.c`](cl_pred.c.md) · [`sv_user.c`](../server/sv_user.c.md) · [`cl_ents.c`](cl_ents.c.md)
**Tier floor** — none

## Purpose

The interface that makes client-side prediction possible. Its whole design principle is that **player movement is a pure
function of a state, a world and a command** — nothing else. So it is expressed as one record holding all three, filled by
whichever side is calling, and the movement code reads nothing outside it.

That purity is the load-bearing decision. Read it before [`pmove.c`](pmove.c.md), and note what is *absent*: no entity
pointers into the server's world, no reference to the game interpreter, no access to time except through the command.

## State

```text
RECORD PlayerMove
  sequence : int                        # for diagnostics only
  # player state, in and out
  origin, angles, velocity : vector
  oldbuttons   : int                   # which buttons were down last command
  waterjumptime : real
  dead, spectator : bool
  # world state, in
  numphysent : int
  physents : PhysEnt[32]               # entry 0 is the world
  # input
  cmd : UserCommand
  # results, out
  numtouch : int ;  touchindex : int[32]

RECORD PhysEnt
  origin : vector
  model  : world model, for a brush entity
  mins, maxs : vector, for a box entity
  info   : int                         # opaque to the mover; the caller's own tag

RECORD MoveVars                        # the tunables, sent by the server
  gravity, stopspeed, maxspeed, spectatormaxspeed
  accelerate, airaccelerate, wateraccelerate
  friction, waterfriction, entgravity

RECORD Trace                           # the result of a sweep
  allsolid, startsolid, inopen, inwater : bool
  fraction : real ;  endpos : vector
  plane : (normal, dist) ;  ent : int
```

**Invariants** —

- **The world is a flat list of at most 32 solid things**, not a tree, not the server's entity array. The caller collects
  whatever could matter — the world model plus nearby movers and players — and hands them over. The mover then treats entry
  zero as the world and the rest uniformly. This is what lets the client, which has only a partial view, run the same code as
  the server.
- The list is **small and fixed**, which bounds the collision cost per movement step and is what makes re-running several
  commands per frame affordable ([`cl_pred.c`](cl_pred.c.md)).
- **Each entry carries an opaque tag** the mover never interprets and hands back in the touch list. The server puts an entity
  number there; the client puts a player slot. That is how one interface serves two owners.
- **The tunables are a record the server sends to the client**, not constants. A client predicting with different gravity or
  acceleration would diverge from the server on every step, so they must travel over the wire
  ([`sv_main.c`](../server/sv_main.c.md) sends them as settings). Freezing them as constants is the most tempting and most
  damaging simplification a rebuild can make.
- **Which buttons were held is part of the state**, because a jump must not repeat while the key is held and the mover is the
  only thing that knows.
- The result includes **what was touched**, because the caller — the server — must run the game logic's touch handlers, and
  the mover must not.

## The operations

**Contract** — run one command's worth of movement over the given world and state; initialize; and the three collision
queries the mover needs: what material is at a point, whether the player fits at a point, and sweep the player's box from one
point to another.

**Invariants** — the three queries are the entire collision surface, and they are implemented over the same precompiled hulls
the server uses ([`pmovetst.c`](pmovetst.c.md), [`world.c`](../server/world.c.md)). That the *client* can answer them is what
makes prediction possible at all, and it is only possible because the collision hulls are part of the map the client also
loads.
