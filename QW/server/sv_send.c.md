# QW/server/sv_send.c

> Assembles and sends each client's packet: the reliable queue folded in, the player's own state, the statistics that changed, the entity snapshot, and a bandwidth choke that skips a client rather than flooding it.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`server.h`](server.h.md) · [`sv_ents.c`](sv_ents.c.md) · [`sv_nchan.c`](sv_nchan.c.md) · [`net_chan.c`](../client/net_chan.c.md) · [`protocol.h`](../client/protocol.h.md)
**Used by** — [`sv_main.c`](sv_main.c.md) calls it once per server frame
**Tier floor** — none

## Purpose

The server's output stage, and the place where the pieces of QuakeWorld's networking meet: the channel's byte budget, the
reliable spillover queue, and the delta snapshot. It also owns the multicast routing — deciding which clients need to hear
about an event — and the console redirection that lets a remotely issued command's output go back to the player who issued it.

## State

```text
VARIABLE sv_redirected : none | to a client | to the console
VARIABLE outputbuf : the redirect accumulator
VARIABLE sv_nailmodel, sv_supernailmodel, sv_playermodel : model indices
VARIABLE sv_phs                                  # potentially hearable set
```

## `SV_SendClientMessages`

**Contract** — once per frame: update the shared reliable content, then for each connected client fold one spillover buffer
into its stream if it fits, drop it if its stream overflowed, skip it if it has not asked for a packet, skip it if its budget
is exhausted, and otherwise send it a full datagram or a bare reliable-only packet depending on whether it has spawned.

```text
FUNCTION send_client_messages()
  update_to_reliable_messages()            # names, scores, settings that changed
  FOR EACH client with a state
    IF it is marked for dropping  drop it ;  CONTINUE
    IF it has a spillover queue AND the head buffer fits in the stream
      append the head buffer ;  shift the queue down by one
    IF its stream overflowed
      clear both buffers ;  announce it ;  drop the client
      force a send with the budget cleared, so the drop message gets out
    IF the client has not sent us a packet since we last replied  CONTINUE
    IF not paused AND the budget forbids a packet
      count a choke ;  CONTINUE
    IF the client has spawned  send_client_datagram(client)
    ELSE                       transmit an empty packet   # carries reliable only
```

**Invariants** —

- **The server replies to a client only when that client has sent something.** The packet rate is therefore set by the
  *client's* frame rate, and a client that stops sending stops being sent to. That is what keeps the server's outbound rate
  proportional to what each client can actually consume, and it is a much better regulator than any server-side rate setting.
- **One spillover buffer per packet, and only if it fits whole.** So the queue drains at one buffer per client per frame, which
  spreads a level-change burst over a few frames.
- **A choked client is skipped entirely, not truncated.** Its snapshot is simply not sent this frame; the next one will be a
  delta from an older reference and will be correspondingly larger, which is self-correcting. Truncating a snapshot would
  corrupt the delta stream.
- **The drop message is sent with the budget cleared**, because a disconnection notice that the budget suppresses leaves the
  player staring at a frozen screen.
- A client that has connected but not spawned is sent **reliable data only**, since it has no world yet — that is how the spawn
  sequence proceeds ([`sv_main.c`](sv_main.c.md)).
- The budget is ignored while the server is paused, matching the channel's own pause handling.

## `SV_SendClientDatagram`

**Contract** — builds one client's unreliable payload: the player's own state and changed statistics, then the entity snapshot,
then whatever the frame accumulated in that client's private datagram, and hands it to the channel. Discards the accumulated
datagram if it will not fit.

**Invariants** — the payload is assembled in a fixed buffer and the **snapshot is written before the accumulated events**, so
that if anything must be dropped it is the events — a lost sound is less harmful than a lost entity update. That priority
ordering is a decision worth copying.

## `SV_WriteClientdataToMessage`, `SV_UpdateClientStats`

**Contract** — write the player's own view information: the punch and velocity applied to their view, the items they hold, their
weapon animation, the ammunition counts, and each statistic whose value differs from what was last sent to this client.

**Invariants** — **statistics are sent only when they change, per client**, with the last sent value remembered per client per
statistic. That is a hand-written delta over a small fixed set, and it is why the health and ammunition display costs nothing
most frames.

A statistic that fits in a byte uses a short form and a larger one uses a long form, chosen per value — so the common case of
small numbers is one byte plus the kind.

## `SV_Multicast`

**Contract** — sends a message to the clients that should receive it, selected by a mode: everyone, everyone who can see a given
point, everyone who can hear it, or the same three with delivery made reliable.

```text
FUNCTION multicast(origin, to)
  determine the leaf containing origin, and from the mode either
    the visible set, the HEARABLE set, or nothing (meaning everyone)
  FOR EACH spawned client
    IF a set applies AND none of the client's leaves is in it  SKIP
    IF the mode is reliable  write into the client's reliable stream
    ELSE                     write into the client's private datagram
```

**Invariants** —

- **There is a hearable set as well as a visible set.** Sound travels around corners and through gaps that light does not, so
  the visibility data is expanded into a second, larger set at map load
  ([`sv_init.c`](sv_init.c.md)) — the union of the visible sets of every visible leaf. Using the visible set for sound makes a
  rocket around a corner silent, which players notice immediately.
- **Every event in the game chooses its own audience**, and the choice is part of the game logic's interface
  ([`pr_cmds.c`](pr_cmds.c.md)). That is how a server with twenty players does not send every explosion to all of them.
- The unreliable variants write into the **client's own datagram**, not a shared one, which is what makes per-client filtering
  possible at all. The original engine has one shared datagram and therefore cannot filter
  ([`sv_main.c`](../../WinQuake/sv_main.c.md)).

## `SV_StartSound`

**Contract** — plays a sound at an entity: assembles the channel, volume, attenuation, position and sound number into a compact
message with a flag byte saying which optional fields are present, and multicasts it to whoever can hear the position.

**Invariants** — volume and attenuation are **omitted when they are the default**, which is most of the time, so the common
sound is a few bytes. The entity and channel share one word, the same packing idea as the entity delta
([`sv_ents.c`](sv_ents.c.md)).

The **attenuation is applied by the client, not the server**: the server sends the coefficient and the position and the client
computes the falloff ([`snd_dma.c`](../client/snd_dma.c.md)). So a sound stays correct as the player moves during its
playback.

## `SV_UpdateToReliableMessages`

**Contract** — once per frame, for each change that every client must learn: a player's score changing, a player's information
string changing, the server's own information changing. Writes each into every client's reliable stream, and clears the change
flags.

**Invariants** — **one pass over the changes, writing to all clients**, rather than each change notifying clients as it happens.
That batching is what keeps a score change from generating a separate write per player per event.

## `SV_ClientPrintf`, `SV_BroadcastPrintf`, `SV_BroadcastCommand`, `SV_PrintToClient`

**Contract** — send text to one client or all clients at a given importance level, and send a console command for every client
to execute.

**Invariants** — text carries an **importance level** and a client can filter below it, so chat and status messages are
separable. The broadcast command facility is how a server makes every client change a setting — powerful and worth noting as a
trust boundary: **the server can execute commands on every client**, which is a capability a rebuild should scope deliberately
rather than inherit.

## `SV_BeginRedirect`, `SV_EndRedirect`, `SV_FlushRedirect`, `Con_Printf`, `Con_DPrintf`

**Contract** — redirect the server's own printing to a client or to a querying address, accumulate it, and flush it in packets;
and the server's print functions, which route to the redirect when one is active.

**Invariants** — **redirection is how a remote administrative command's output reaches the person who typed it**
([`sv_ccmds.c`](sv_ccmds.c.md)). The accumulator is flushed when it nears a packet's worth, so long output arrives in several
pieces rather than being truncated.

The server's print function is *this* file's, not the client's — the two programs have different ones with the same name, which
is how one body of shared code prints correctly in both ([`bothdefs.h`](../client/bothdefs.h.md)).

## `SV_FindModelNumbers`, `SV_SendMessagesToAll`

**Contract** — look up the model indices the special-cased encodings need
([`sv_ents.c`](sv_ents.c.md)); and force a send to every client regardless of whether they asked, used when something must go
out now.

**Invariants** — the nail encoding identifies its entities **by model index**, which is assigned at map load, so the indices
must be found after the map is loaded and before any frame is sent.

**Notes** — the shape of this file is the answer to "how does a server serve twenty players on a modem": reply only when
replied to, filter every event by audience, delta everything, and skip a client rather than flooding it. None of the four is
complicated and all four are necessary.
