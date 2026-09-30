# QW/server/sv_phys.c

> Entity physics with the player's movement taken out: pushers that move riders and refuse when crushed, tossed objects, stepping monsters, and a special pass so a projectile fired this frame still moves this frame.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`server.h`](server.h.md) · [`world.h`](world.h.md) · [`pr_exec.c`](pr_exec.c.md)
**Used by** — [`sv_main.c`](sv_main.c.md) each frame; [`sv_user.c`](sv_user.c.md) for the projectile pass
**Tier floor** — none

## Purpose

Read [`sv_phys.c`](../../WinQuake/sv_phys.c.md) first: the movement types, the slide move, the bounce, the think scheduling, the
water and gravity handling and the pusher's rider logic are all the same and are the load-bearing content. This file is a third
shorter, and what was removed is as informative as what was added.

## State

As [`sv_phys.c`](../../WinQuake/sv_phys.c.md); the records are unchanged except where **What differs** says otherwise.

## What was removed, and why

**The whole player movement path is gone**: the walk move, the wall friction, the stuck check, the water check and the
client physics type. All of it moved to [`pmove.c`](../client/pmove.c.md) so the client could run it. A player entity is now
driven entirely by [`sv_user.c`](sv_user.c.md), which calls the shared mover and writes the result back.

That is the single largest structural difference between the two engines, and it is visible here as an absence.

**The follow movement type is gone**, and with it the ability for one entity to be attached to another's motion by the engine.
Game logic that wants it does it itself.

**Rotating pushers are gone.** The original has a separate path for a pusher that turns, which moves riders through an angular
transform and is the source of several of its worst bugs. QuakeWorld deletes it: a pusher translates only. That is a deliberate
simplification of the *game*, not only of the code, and a rebuild should know it is choosing.

## What was added

## `SV_Push`, `SV_PushMove`

**Contract** — move a pusher by a displacement, carrying everything resting on it and refusing the whole move if something would
be crushed; and drive that from a pusher's velocity and the frame time.

```text
FUNCTION push(pusher, move) -> bool
  move the pusher and relink it
  FOR EACH entity that was inside the pusher's new bounds
    IF it is not affected by pushing  SKIP
    remember its position
    try to move it by the same displacement, sliding
    IF it is no longer blocked  relink it ;  record it as moved ;  CONTINUE
    IF it is resting on the pusher  carry it and CONTINUE
    # it is genuinely blocked: undo EVERYTHING
    restore the pusher and every entity moved so far
    call the game logic's blocked handler on the pusher
    RETURN false
  RETURN true
```

**Invariants** —

- **The push is all-or-nothing.** Restructuring the original's push into a function that *returns whether it succeeded* and
  restores every partial move on failure is the change, and it fixes a real class of bug: in the original, a door blocked
  part-way through its rider list leaves some riders moved and some not. A rebuild should adopt the transactional form.
- The blocked handler is the game logic's, so what happens when a door is obstructed — reversing, crushing, waiting — is a game
  decision.
- **Only translation.** See above.

## `SV_RunEntity`, `SV_ProgStartFrame`, `SV_Physics`

**Contract** — run one entity's physics once per frame, marked so it cannot run twice; call the game logic's start-of-frame
function; and the frame pass over every entity.

**Invariants** — **an entity records which frame it last ran in and refuses to run twice.** That is needed because the projectile
pass below can run an entity out of turn, and running physics twice in one frame doubles an object's speed. The guard is one
comparison and its absence is an intermittent bug.

## `SV_RunNewmis`

**Contract** — immediately after a player's command, if the game logic created a projectile during it, run that projectile's
physics for the remainder of the frame.

**Invariants** — **a projectile fired this frame must move this frame.** Without this, a rocket spends one frame at the muzzle,
and at a player's own frame rate that is a visible and tactically significant delay — it makes point-blank shots behave
differently from distant ones. The game logic signals the new projectile through a dedicated global
([`progdefs.h`](progdefs.h.md)), which is why that global exists.

This is a small addition with a large effect on how the game feels, and it is the kind of thing that only appears once movement is
being predicted and the latency of everything else becomes noticeable by comparison.

## `SV_AddGravity`

**Contract** — as the original, but takes a **scale factor**.

**Invariants** — gravity is now per entity, scaled by a field the game logic sets, and for a player it is the per-player override
([`server.h`](server.h.md)) that must also be sent to the client so its prediction matches
([`pmove.h`](../client/pmove.h.md)).

## `SV_SetMoveVars`

**Contract** — publishes the movement tunables into the server's public dictionary so they reach every client.

**Invariants** — **the tunables must be published whenever they change**, not only at connection, or a client that joined before a
change predicts with the old values. This function is the reason the tunables live in the dictionary rather than as plain
settings ([`sv_ccmds.c`](sv_ccmds.c.md)).

## `SV_Physics_Pusher`, `SV_Physics_None`, `SV_Physics_Noclip`, `SV_Physics_Step`, `SV_Physics_Toss`, `SV_RunThink`, `SV_CheckVelocity`, `SV_FlyMove`, `SV_PushEntity`, `SV_Impact`, `SV_ClipVelocity`, `SV_CheckWaterTransition`, `SV_AddGravity`

**Contract** — the remaining movement types and the shared helpers, unchanged in substance from
[`sv_phys.c`](../../WinQuake/sv_phys.c.md). Read that twin.

**Notes** — the ordering fact from [`sv_main.c`](sv_main.c.md) belongs here too: **the world is simulated before player commands
are run**, so a player acts on a world that has already moved. The design notes name this as an intended change
([`newnet.txt`](newnet.txt.md)), and it is why a rocket a player sees is at a position they can actually shoot at.
