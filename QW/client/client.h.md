# QW/client/client.h

> The client's state: a ring of frames each holding the command sent, the snapshot received and the timing, plus every player's information and the predicted present used for rendering.

**Needs** — [`protocol.h`](protocol.h.md) · [`net.h`](net.h.md) · [`pmove.h`](pmove.h.md) · [`render.h`](render.h.md)
**Used by** — every `cl_*.c`, and the renderer and interface files that read the view
**Tier floor** — none

## Purpose

Read [`client.h`](../../WinQuake/client.h.md) for the shared parts: the entity list, the dynamic lights, the temporary entities, the
sound state and the view fields. The differences are the whole of the prediction and delta machinery, expressed as data.

## State

As [`client.h`](../../WinQuake/client.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

## The frame ring

```text
RECORD Frame                        # one per outgoing packet sequence
  senttime      : time              # when we sent this packet
  receivedtime  : time              # when the reply came, or "lost", or "choked"
  playerstate[] : each player's reported state, with the time it was valid at
  cmd           : the command we sent in this packet
  invalid       : bool              # its delta reference was unusable
  delta_sequence: which snapshot we asked it to delta from
  packet_entities : the snapshot as applied
VARIABLE frames[ring size]
VARIABLE cl.parsecount, cl.validsequence
```

**Invariants** —

- **One ring, indexed by the outgoing packet sequence, holding both what we sent and what came back for it.** That single structure
  is the client's whole memory: prediction replays commands from it ([`cl_pred.c`](cl_pred.c.md)), the delta decompression finds its
  reference in it ([`cl_ents.c`](cl_ents.c.md)), the latency and loss measurement reads its timings
  ([`cl_parse.c`](cl_parse.c.md)), and the diagnostic display draws it ([`gl_ngraph.c`](gl_ngraph.c.md)). Four features, one array.
- **The reply time doubles as an outcome marker**, with distinguished values for lost and choked. Overloading a timestamp is
  incidental; the three-way distinction is not ([`cl_parse.c`](cl_parse.c.md)).
- **The ring's size is the maximum prediction horizon and the maximum delta age**, and the server's ring is the same size
  ([`server.h`](../server/server.h.md)). The two must agree or one side offers a reference the other has discarded.
- **Each player's state carries the time it was valid at**, not the time it arrived
  ([`cl_ents.c`](cl_ents.c.md)).

## The predicted present

```text
VARIABLE simorg, simvel, simangles   # where we believe we are, for rendering
VARIABLE viewangles                  # what the player is aiming, never predicted
VARIABLE cl.time                     # the moment being displayed
VARIABLE cls.latency                 # smoothed round trip
```

**Invariants** — **the rendered position is a separate field from any received position.** Nothing overwrites it but the prediction,
and nothing else reads it but the renderer and the sound. Keeping the predicted present distinct from the authoritative past is what
makes it possible to turn prediction off and compare
([`cl_pred.c`](cl_pred.c.md)).

The displayed time is deliberately behind the present, and both the local player and everyone else are drawn at it
([`cl_ents.c`](cl_ents.c.md)).

## Per-player information

```text
RECORD PlayerInfo
  userid, userinfo (the dictionary), name, topcolor, bottomcolor
  frags, entertime, ping
  spectator : bool
  skin : a reference to the shared skin entry
  translations : the colour remap table
```

**Invariants** — **every client holds every player's dictionary**, which is what lets it fetch their skin
([`skin.c`](skin.c.md)) and draw their name. A change to one entry costs a few bytes
([`cl_parse.c`](cl_parse.c.md)).

## Connection state

```text
VARIABLE cls.state : disconnected | connecting | on the server | active
VARIABLE cls.servername, cls.qport, cls.netchan
VARIABLE the download and upload state
VARIABLE the recording state
VARIABLE cls.spectator, cl.spectator
```

**Invariants** — **four connection states, and the third exists only so the first frame can be recognized**: the transition to
active happens when the first snapshot is applied ([`cl_pred.c`](cl_pred.c.md)), because that is the first moment a frame can be
drawn. Entering the game is defined by having something to render, not by a handshake completing.

## What was removed

**Every field relating to a local server** — the original's client state knows whether the server is in the same process
([`client.h`](../../WinQuake/client.h.md)) and takes shortcuts when it is. There is no local server now, so those vanish along with
the shortcuts.

**The saved-game and single-player campaign fields.**

**The entity array as persistent state** — entities now live in the frame ring's snapshots and the visible list is rebuilt each
frame ([`cl_ents.c`](cl_ents.c.md)).

**Notes** — the shape to take from this file is the first invariant: **one ring indexed by packet sequence, holding the command sent
and the state received, serves prediction, delta decompression, latency measurement and diagnostics at once.** Designing that ring
first makes the rest of the client fall out of it.
