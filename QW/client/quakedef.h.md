# QW/client/quakedef.h

> The client program's umbrella header: everything it compiles against, in dependency order, plus the startup parameters.

**Needs** — [`bothdefs.h`](bothdefs.h.md) and, through it, every other header in the client
**Used by** — every `.c` file in this directory
**Tier floor** — none

## Purpose

Read [`quakedef.h`](../../WinQuake/quakedef.h.md) for what the original's umbrella does. The differences are two.

## State

As [`quakedef.h`](../../WinQuake/quakedef.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**The shared constants moved out** into [`bothdefs.h`](bothdefs.h.md), so that the server can include exactly the same ones. What
remains here is the client's own include list and the startup parameters.

**The server's headers are gone**: no server state, no entity pool, no game-logic interpreter, no world. A client compiles against
the protocol, the movement model and the renderer, and knows nothing about how the world is simulated. That absence is the
architectural fact — **the original's client can see the server's data structures and this one cannot** — and it is what made the
client-server split real rather than nominal.

**The movement model and its collision queries are included**
([`pmove.h`](pmove.h.md)), which is the one piece of simulation the client does hold.

**Invariants** — the include order is the dependency order and is load-bearing; it is also the reading order for this chapter.

**Notes** — comparing the two umbrella headers is the fastest way to see what each program is. A rebuild should keep the property
that the client's include list cannot reach the server's.
