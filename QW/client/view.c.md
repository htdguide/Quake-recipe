# QW/client/view.c

> The camera: view height and bob, the roll from sideways motion, the damage and liquid tints, and the view punch, all computed from the predicted position.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`render.h`](render.h.md) · [`cl_cam.c`](cl_cam.c.md)
**Used by** — [`cl_main.c`](cl_main.c.md) each frame; the renderers read the result
**Tier floor** — none

## Purpose

Read [`view.c`](../../WinQuake/view.c.md) for the whole substance: the bob derived from horizontal speed, the roll from sideways
velocity, the idle sway, the damage roll and pitch from where the hit came from, the tint accumulation and the drift back to level.

## State

As [`view.c`](../../WinQuake/view.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**The view is built from the predicted position, not the received one**
([`cl_pred.c`](cl_pred.c.md)), so the camera is where the player believes they are. That is the one change and it is the whole point.

**The view punch is decayed locally** and is triggered by two dedicated message kinds rather than a transmitted value
([`protocol.h`](protocol.h.md)). The original transmits the punch each frame; here the server says "kick, small" once and the client
runs the decay. The design notes list this change explicitly as an intention
([`newnet.txt`](../server/newnet.txt.md)) — *make the punch a client-side operation* — and the reason is that a transmitted decay costs
bytes every frame for something the client can compute.

**The spectator camera overrides the view** when spectating ([`cl_cam.c`](cl_cam.c.md)).

**Invariants** — **anything that decays predictably should be triggered, not transmitted.** That is the transferable rule, and the view
punch is its clearest instance in the engine: one message replaced a per-frame field.

The tint and bob are computed from the *displayed* time
([`client.h`](client.h.md)), not the server's, so they are smooth at any frame rate.
