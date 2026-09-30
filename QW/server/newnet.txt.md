# QW/server/newnet.txt

> Data: the author's working notes while designing QuakeWorld's networking — the task list, the packet contents as first sketched, and the prediction loop in four lines.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The design document for [`net_chan.c`](../client/net_chan.c.md), [`sv_ents.c`](sv_ents.c.md) and
[`cl_pred.c`](../client/cl_pred.c.md), written before them. It is the most valuable non-source file in the whole recipe, because
it states the design's intent in the author's own terms and shows which parts were planned and which emerged.

## State

Data; no run-time state.

## What it records

**The prediction loop, in four lines**, and the shipped implementation is exactly this:

```text
build movement
send the movement packet to the server
set the player position to the last known good position
execute all unacknowledged movement packets
```

That is [`cl_pred.c`](../client/cl_pred.c.md) in full. The note beside it — *"needs to know all things that can affect
movement"* — is the requirement that produced [`pmove.h`](../client/pmove.h.md)'s self-contained record, and the phrase
*"movement context"* with its list of origin, velocity and size is that record's first draft.

**The two packet contents as first designed:**

```text
client to server:  last good server time received, milliseconds since the last
                   move frame, angles, movement, button states, impulse
server to client:  last move message received, origin, velocity
```

Compare the shipped forms ([`protocol.h`](../client/protocol.h.md)): the client half survived nearly verbatim; the server half
grew into the delta snapshot, because sending only the player's own origin and velocity was not enough to draw the world.

**The channel's header, sketched as three fields** — a sequence, a reliable sequence and a reliable payload bit — which became
the two-word form with the reliable state packed into the top bits
([`net_chan.c`](../client/net_chan.c.md)). The note shows the reliable acknowledgement was going to be a full sequence and was
reduced to one bit.

**A task list that names problems the shipped code solves**, and a few it does not:

- *"problem with reconnect to same server out of order packets"* — answered by the out-of-band connection protocol and the
  connection identifier ([`net_chan.c`](../client/net_chan.c.md)).
- *"worry about partial connection during level change"*, *"client zombie state?"* — the spawn state machine in
  [`sv_main.c`](sv_main.c.md).
- *"central clearinghouse for active servers"* — the directory announcement in [`sv_ccmds.c`](sv_ccmds.c.md).
- *"allow remote locations to subscribe to console output"* — the redirection in [`sv_send.c`](sv_send.c.md).
- *"make punchangle a client side operation"* — done; the view kick is computed by the client
  ([`view.c`](../client/view.c.md)).
- *"simulate world first, then clients"* — the ordering in [`sv_phys.c`](sv_phys.c.md).
- *"packet filter list"*, *"separate accept/reject list for server and client packets"* — implemented as the address filter in
  [`sv_main.c`](sv_main.c.md).
- *"remove sys_printf"*, *"remove host_client"* — **not** done; both survive in the shipped code
  ([`sv_send.c`](sv_send.c.md), [`sv_ccmds.c`](sv_ccmds.c.md)).
- *"does not address interpolating other objects' movement"* — the note's own closing admission, and the gap was filled by
  sending each player's last command and the time since it ran ([`sv_ents.c`](sv_ents.c.md),
  [`cl_ents.c`](../client/cl_ents.c.md)).

**A sketch of a ring of sent move messages with their times**, capped at 64, which is the frame ring in
[`client.h`](../client/client.h.md).

**Notes** — a rebuilder should read this file before writing any networking, because it shows the order the problems arrive in:
make movement a pure function, then predict it, then discover that other players still need interpolating, then discover that the
server's half of the packet must become a delta snapshot. Arriving at the same four conclusions in the same order is faster than
deriving the finished design from the finished code.

The unfinished items are equally useful: they mark where the shipped code is known by its author to be untidy.
