# QW/client/net_chan.c

> The sequenced channel: one packet per frame carrying a 64-bit header, a reliable stream acknowledged by a single alternating bit, unreliable data appended when it fits, and an outgoing byte budget that paces the sender to the link.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net.h`](net.h.md) · [`common.h`](common.h.md) · [Seam: Unreliable datagram transport](../../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`cl_main.c`](cl_main.c.md) · [`sv_main.c`](../server/sv_main.c.md) · [`sv_send.c`](../server/sv_send.c.md) · [`sv_nchan.c`](../server/sv_nchan.c.md) — **both** programs compile this one file
**Tier floor** — none

## Purpose

The replacement for everything in [`net_dgrm.c`](../../WinQuake/net_dgrm.c.md), and the most important single file in the
QuakeWorld chapter. The original's reliability layer treats the link as a message pipe: it sends a message, waits for an
acknowledgement, retransmits on a timer, and refuses to send while one is outstanding. That is correct and unusable over the
internet, because the player's movement waits behind it.

This file replaces it with a design where **the packet rate is fixed by the frame rate and nothing ever waits**. One packet
goes out per frame; it always carries the current unreliable state; the reliable stream rides along when there is room. A
lost packet is not retransmitted — the next one supersedes it. Only the reliable stream is retransmitted, and its
acknowledgement costs one bit.

Read this twin, [`sv_ents.c`](../server/sv_ents.c.md) and [`pmove.c`](pmove.c.md) together: they are the three ideas that
make a 1996 engine playable over a modem, and every networked game since has used all three.

## State

```text
CONSTANT packet_header = 8 bytes            # two 32-bit words; a client adds 2 more
CONSTANT max_backup = 200                   # packets of credit the budget allows
CONSTANT max_latent                         # ring size for the sent-packet log
CONSTANT old_avg                            # smoothing weight for the averages

RECORD Netchan
  fatal_error        : bool
  last_received      : time
  remote_address     : address ;  qport : 16-bit
  # outgoing
  outgoing_sequence      : int
  reliable_sequence      : 0 or 1           # the alternating bit, NOT a counter
  last_reliable_sequence : int              # which outgoing packet carried it
  message     : buffer  (the reliable stream being composed)
  reliable_buf, reliable_length             (the copy in flight)
  # incoming
  incoming_sequence, incoming_acknowledged : int
  incoming_reliable_sequence     : 0 or 1
  incoming_reliable_acknowledged : 0 or 1
  # pacing and statistics
  rate, cleartime : real
  outgoing_size[], outgoing_time[] : ring, indexed by sequence
  frame_latency, frame_rate : real ;  drop_count, good_count : int
VARIABLE net_drop                            # how many packets the last read skipped
```

## The packet header

```text
word 1:  31 bits  outgoing sequence
          1 bit   this packet carries reliable data
word 2:  31 bits  highest sequence received from the peer
          1 bit   the peer's reliable alternating bit, as last seen
[client only] 16 bits  a port identifier chosen by the client
```

**Invariants** —

- **The reliable stream is acknowledged by one alternating bit, not by a sequence number.** The sender flips its bit each
  time it starts a new reliable payload; the receiver echoes back the bit of the last reliable payload it accepted. The
  sender concludes a loss when the peer has acknowledged a *packet* later than the one that carried the payload, but is
  still echoing the *previous* bit. One bit suffices because **only one reliable payload is ever in flight**, which is the
  same restriction as the original — kept, because it removes all the bookkeeping, and harmless, because the reliable stream
  carries only things that are not urgent.
- **A sequence of all ones marks a packet as out of band**, to be handled without a channel at all. That is how connection
  requests, server queries and rejections are sent — there is no handshake state machine
  ([`sv_main.c`](../server/sv_main.c.md)).
- **The port identifier exists because address translation rewrites the client's source port mid-game.** A channel is matched
  on the address's host part plus this identifier, and the port is then *updated* to whatever the packet came from. Without
  it, a player behind such a router is disconnected at a random moment for no visible reason. This is a real-world
  protocol requirement that nothing in a laboratory reveals, and a rebuild should include an equivalent connection
  identifier from the start.
- The identifier is chosen **randomly at startup** from the clock and the process identity, so two clients behind one router
  differ.

## `Netchan_Transmit`

**Contract** — sends one packet: decide whether the reliable payload must be resent, promote the composed reliable stream
into the in-flight buffer if that buffer is free, write the header, append the reliable payload if it is being sent, append as
much of the caller's unreliable data as fits, send, and charge the packet against the byte budget. A zero-length unreliable
part still sends a packet.

```text
FUNCTION transmit(chan, data)
  IF the composing buffer has overflowed  mark the channel fatally broken ;  RETURN
  send_reliable = false
  IF the peer has acknowledged a packet LATER than the one that carried our payload
     AND the peer's echoed bit still differs from ours
    send_reliable = true                       # it was lost: resend the same bytes
  IF nothing is in flight AND the composing buffer has content
    copy the composing buffer into the in-flight buffer ;  empty the composing buffer
    FLIP our alternating bit
    send_reliable = true
  word1 = outgoing_sequence WITH the reliable flag in the top bit
  word2 = incoming_sequence WITH the peer's-reliable bit in the top bit
  outgoing_sequence += 1
  write word1, word2 [, the port identifier if this is a client]
  IF send_reliable
    append the in-flight buffer ;  last_reliable_sequence = outgoing_sequence
  IF the remaining room is at least the unreliable length  append it
  send the packet
  log its size and time in the ring, indexed by sequence
  charge it: cleartime = MAX(cleartime, now) + size * rate
```

**Invariants** —

- **Reliable data is always placed first and the unreliable part is dropped if it does not fit.** So the reliable stream can
  never be starved by movement updates, and a frame's worth of entity state can be lost without consequence. The priority is
  the right way round and it is the whole reason the two share a packet.
- **The receiver is told nothing about the split.** It reads the packet as one message. So the reliable stream carries no
  framing of its own, and a reliable payload must therefore be a whole number of protocol messages
  ([`sv_nchan.c`](../server/sv_nchan.c.md) enforces that on the server side).
- **A retransmission sends the identical bytes**, and does not happen again until a packet sent *after* the retransmission
  has been acknowledged and the bit still disagrees. Without that second condition a slow link would retransmit every frame
  and collapse.
- An overflow of the composing buffer is **fatal to the channel**, because the stream cannot skip content and stay a stream.
  That is why the server's reliable writer must check for room before writing anything
  ([`sv_nchan.c`](../server/sv_nchan.c.md)).
- The sent-packet ring is indexed by sequence masked to its size, so it holds the last few packets' sizes and times — used
  for the statistics below, and intended for a rate estimator that is present but disabled.

## `Netchan_CanPacket`, `Netchan_CanReliable`

**Contract** — report whether the byte budget allows another packet, and whether a new reliable payload may be composed.

```text
FUNCTION can_packet(chan)
  RETURN cleartime < now + max_backup * rate
FUNCTION can_reliable(chan)
  IF something is already in flight  RETURN false
  RETURN can_packet(chan)
```

**Invariants** —

- **The budget is a virtual clock, not a counter.** Every packet pushes `cleartime` forward by its size times the seconds
  per byte the link is assumed to sustain. A sender may run ahead of real time by a fixed credit and no further. This is a
  leaky bucket, and it is the mechanism by which a server with twenty players on a modem link each does not simply flood
  them.
- **The rate is per channel and negotiable**, defaulting to a modem-era figure, and the player sets it
  ([`cl_main.c`](cl_main.c.md) sends it as a setting; [`sv_main.c`](../server/sv_main.c.md) clamps it). Getting this wrong
  in either direction is the classic symptom: too high and the link drops packets in bursts, too low and the player sees
  stale entities.
- The credit of two hundred packets is generous on purpose: it lets a burst through — a level change, a full entity update —
  and then throttles.
- The budget is **reset while the server is paused**, so a pause does not accumulate credit that floods on resume.

## `Netchan_Process`

**Contract** — validates an arriving packet and consumes its header, leaving the payload for the caller to read. Reports
whether the packet should be used.

```text
FUNCTION process(chan) -> bool
  IF the packet did not come from this channel's address  REJECT
  read word1, word2 [, the port identifier if this is a server]
  split off the two top bits
  IF sequence <= our recorded incoming sequence  REJECT      # stale or duplicate
  net_drop = sequence - (incoming_sequence + 1)              # how many were lost
  IF net_drop > 0  count a drop
  IF the peer's echoed bit equals our alternating bit  the in-flight buffer is free
  incoming_sequence = sequence
  incoming_acknowledged = sequence_ack
  incoming_reliable_acknowledged = the peer's echoed bit
  IF this packet carried reliable data  FLIP our incoming bit
  update the smoothed latency and packet-rate averages
  last_received = now
  ACCEPT
```

**Invariants** —

- **Out-of-order and duplicate packets are dropped, not reordered.** There is no receive buffer. A packet older than the
  newest one seen carries nothing useful, because every unreliable payload is a complete snapshot of the present.
  *This is the property that the whole design rests on*: make every unreliable message self-contained, and a reordering
  network needs no reordering code.
- **A gap is reported to the caller, not hidden.** The count of skipped packets is published, and the client uses it to know
  that its predicted movement has lost its corrections ([`cl_pred.c`](cl_pred.c.md)) and the server uses it to know a
  player's commands were lost ([`sv_user.c`](../server/sv_user.c.md)). Losing packets is a normal condition with a visible
  consequence, not an error.
- The **acknowledgement is implicit**: word two of every packet acknowledges, so there are no acknowledgement packets at all.
  On a symmetric exchange at the frame rate that is free.
- The source-address check plus the narrow window of acceptable sequence numbers is the **only** protection against a forged
  packet, and the source notes it as such. It is weak — anyone who can observe the traffic can forge — and a rebuild
  handling untrusted networks needs a real authenticator on the channel. The recipe records the original's position
  honestly: the channel is not authenticated.
- A dropped packet **does not break the reliable stream**, because its acknowledgement bit is carried by every subsequent
  packet.

## `Netchan_Setup`, `Netchan_Init`

**Contract** — clear a channel, point its composing buffer at its own storage, allow that buffer to signal overflow rather
than fail hard, record the peer and the port identifier, and set the default rate; and at startup, pick the random port
identifier and register the diagnostic settings.

**Invariants** — the composing buffer is marked as **allowed to overflow**, meaning an overrun sets a flag instead of
aborting, precisely so the channel can report the fatal condition itself in the transmit path.

## `Netchan_OutOfBand`, `Netchan_OutOfBandPrint`

**Contract** — send a datagram with the out-of-band marker in place of a sequence, with raw bytes or with formatted text.

**Invariants** — **the entire connection protocol is out-of-band text.** A client asks to connect with a formatted string; the
server replies with a string; a query for the server's status is answered with a string. That means a connection needs no
state on the server until it succeeds, which is what makes the server immune to a half-open connection flood, and it means
the protocol is inspectable with any tool that can send a datagram. It is one of the best decisions in QuakeWorld and a
rebuild should copy it.

**Notes** — the rate estimator that would infer the link's capacity from the round-trip time of small packets is written and
**disabled**. It is worth reading as a warning: the measurement is confounded by the fixed packet rate, and the author left
it off rather than ship a wrong estimate. A rebuild wanting congestion control should use a real algorithm, not this.

The statistics — smoothed latency in packets and smoothed inter-packet interval — are what the player's connection display
shows ([`sbar.c`](sbar.c.md)). Latency is measured in **packets outstanding**, not in seconds, which is the natural unit here
because the packet rate is the frame rate.
