# QW/server/sv_move.c

> Monster locomotion: step toward a goal, refuse a step that would fall too far or land in liquid, and try both diagonals when a direct step fails.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`server.h`](server.h.md) · [`world.h`](world.h.md)
**Used by** — [`pr_cmds.c`](pr_cmds.c.md) exposes it to the game logic
**Tier floor** — none

## Purpose

Substantially identical to [`sv_move.c`](../../WinQuake/sv_move.c.md) — read that twin for the step-and-check algorithm, the
ground-required rule, the drop-off limit and the diagonal retry, all of which are the load-bearing content.

## State

As [`sv_move.c`](../../WinQuake/sv_move.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Only the removal of a debug path and the adjustment to call the local trace interface. Monster movement was not touched by the
networking rewrite, because a monster is simulated on the server and never predicted.

**Notes** — that this file is unchanged is itself worth recording: the client-prediction work touched only the *player's*
movement, and every other moving thing in the game kept its original physics. A rebuild can adopt prediction without revisiting
its artificial-intelligence movement at all.
