# QW/server/sv_nchan.c

> A queue of overflow buffers in front of each client's reliable stream, so that a burst of reliable messages is spread over several packets instead of dropping the player.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`server.h`](server.h.md) · [`net_chan.c`](../client/net_chan.c.md) · [`common.h`](../client/common.h.md)
**Used by** — [`sv_send.c`](sv_send.c.md) · [`sv_ccmds.c`](sv_ccmds.c.md) · [`sv_user.c`](sv_user.c.md) · [`sv_main.c`](sv_main.c.md) — every reliable write to a client goes through it
**Tier floor** — none

## Purpose

Solves a problem the channel ([`net_chan.c`](../client/net_chan.c.md)) creates. The channel's composing buffer is one packet's
worth, and overflowing it is fatal to the connection. But the server legitimately generates bursts of reliable data — twenty
players' names and scores at a level change, a flurry of chat, a spawn sequence — that exceed one packet.

The answer here is a **queue of spillover buffers**: when the composing buffer has no room for the message about to be written,
subsequent writes go into a spillover buffer instead, and [`sv_send.c`](sv_send.c.md) folds one spillover buffer into the
composing buffer per packet as room appears. The channel never sees an overflow, and the reliable stream stays a stream.

The important design point is *where* the check happens, and it is the reason every reliable write in the server goes through
this file rather than touching the buffer directly.

## State

```text
CONSTANT max_back_buffers
# per client (in server.h):
VARIABLE backbuf : buffer                      # the one currently being written
VARIABLE backbuf_data : byte[max][packet size] # the queue's storage
VARIABLE backbuf_size : int[max]               # how much is in each
VARIABLE num_backbuf : int                     # 0 means write straight to the channel
```

**Invariants** — **a queue depth of zero means "write directly to the channel"**, so the common case costs one comparison and
nothing else. The queue only comes into existence under pressure.

## `ClientReliableWrite_Begin`

**Contract** — begins one reliable message: given the message kind and an **upper bound on the message's total size**, decides
whether it fits, rotating to a spillover buffer if not, then writes the kind byte.

**Invariants** — **the caller must state the message's maximum size before writing any of it.** That is the whole contract, and
it is what makes the scheme work: the decision to spill is made once, before the first byte, so a message is never split across
two buffers. A message split in the middle would be unparseable, because the receiver sees the reliable stream as one
undifferentiated byte sequence ([`net_chan.c`](../client/net_chan.c.md)).

Every call site therefore carries a size estimate, and **an estimate that is too small is a latent corruption**, not an
overflow: the writes will silently cross a buffer boundary. That is the one real hazard in the design, and a rebuild should
either compose each message in a scratch buffer and append it whole — which removes the need for the estimate entirely — or
assert the estimate at the finish.

## `ClientReliableCheckBlock`

**Contract** — if the queue is non-empty, or the composing buffer cannot hold the stated size, ensure a spillover buffer is
available: create the first one, or advance to the next when the current is full. Exhausting the queue marks the client's
stream as overflowed, which drops them.

```text
FUNCTION check_block(client, maxsize)
  IF the queue is empty AND the channel's buffer has room for maxsize  RETURN
  IF the queue is empty
    start spillover buffer 0 ;  num_backbuf = 1
  IF the current spillover buffer cannot hold maxsize
    IF the queue is at its limit
      warn ;  clear the current buffer ;  mark the stream overflowed ;  RETURN
    start the next spillover buffer ;  num_backbuf += 1
```

**Invariants** —

- **Once anything is queued, everything goes to the queue** until it drains, even if the composing buffer has room. Order must
  be preserved, and letting a later small message overtake a queued one would reorder the stream.
- **Exhausting the queue drops the client**, by marking the channel's buffer as overflowed — reusing the channel's own fatal
  condition rather than inventing a second one. A player who cannot be kept up with is disconnected, which is the correct and
  honest outcome; the alternative is silently losing reliable content, which corrupts their view of the game permanently.
- The spillover buffers are marked as **allowed to overflow** so that an under-estimated size sets a flag rather than aborting
  the server; the finish step below turns that flag into a disconnection.

## `ClientReliable_FinishWrite`

**Contract** — records the current spillover buffer's length, and if it overflowed, marks the client's stream as overflowed.

**Invariants** — called after **every** primitive write, not once per message, which is why it is cheap and why the recorded
length is always current — [`sv_send.c`](sv_send.c.md) may fold a buffer into a packet at any time.

## `ClientReliableWrite_Byte`, `_Char`, `_Short`, `_Long`, `_Float`, `_Angle`, `_Angle16`, `_Coord`, `_String`, `_SZ`

**Contract** — write one value of each protocol type, to the current spillover buffer if the queue is non-empty, otherwise
straight to the channel's composing buffer. Each finishes the write when it went to a spillover buffer.

**Invariants** — ten near-identical functions differing only in which encoder they call. The repetition is incidental; a rebuild
writes one function parameterized by the encoder, or composes into a scratch buffer and appends once.

What is *not* incidental is that **every reliable write in the server goes through one of these ten**. No code anywhere writes
to a client's channel buffer directly. That discipline is what makes the spillover invisible, and a single direct write
somewhere would break it silently.

**Notes** — the two angle encoders reflect the protocol's two angle resolutions: one byte for most purposes and two where
precision matters ([`protocol.h`](../client/protocol.h.md)). See [`common.c`](../client/common.c.md) for the byte encoder's
rounding quirk, which must be reproduced exactly.
