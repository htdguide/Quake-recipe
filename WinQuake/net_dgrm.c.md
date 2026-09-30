# WinQuake/net_dgrm.c

> The reliability layer: turns unreliable datagrams into one ordered, acknowledged, fragmented reliable stream per direction, plus a sequence-numbered unreliable stream — and implements the connection handshake and server discovery.

**Needs** — [`net.h`](net.h.md) · [`net_dgrm.h`](net_dgrm.h.md) · [`common.h`](common.h.md) · [`quakedef.h`](quakedef.h.md) · [`server.h`](server.h.md) and [`client.h`](client.h.md) (for the server information a discovery reply carries) · [`cvar.h`](cvar.h.md) · [`cmd.h`](cmd.h.md) · [Seam: Unreliable datagram transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`net_main.c`](net_main.c.md) registers it as a message driver
**Tier floor** — none

## Purpose

This is the file a rebuild must implement rather than shop for. Everything above it — the protocol, the server's
send logic, the client's decode — assumes exactly this behaviour, and two engines of this generation only
interoperate if they agree on it.

The model is deliberately minimal: **one outstanding reliable message per direction**, fragmented into
datagram-sized pieces, each piece acknowledged before the next is sent. Not a window, not a queue. Plus an
independently sequenced unreliable stream where out-of-order arrivals are simply dropped.

## State

Per connection, from [`net.h`](net.h.md): a send sequence, an acknowledge sequence, an unreliable send sequence, the
outstanding reliable message held in full for retransmission, a receive sequence, an unreliable receive sequence, a
reassembly buffer, and two flags — whether a reliable message may be started, and whether the next fragment is
pending.

```text
VARIABLE packetBuffer : { length : int (32-bit, BIG-endian)
                          sequence : int (32-bit, big-endian)
                          data : byte[1024] }
VARIABLE packetsSent, packetsReSent, packetsReceived : int
VARIABLE receivedDuplicateCount, shortPacketCount, droppedDatagrams : int
```

**Invariants** — **the header is big-endian**, unlike every other multi-byte value in the engine
([`net.h`](net.h.md)). The length field includes the header and shares its word with the flags.

## `Datagram_SendMessage`

**Contract** — takes a connection and a buffer; copies the whole message into the connection's retransmission
store, sends its first fragment with a fresh sequence number and the end-of-message flag when it fits entirely, and
marks the connection unable to accept another reliable message. Returns 1, or −1 if the transport failed.

```text
FUNCTION datagram_send_message(sock, data) -> int
  copy the whole message INTO sock.sendMessage ;  record its length
  IF the message fits in one datagram
    dataLen = its length ;  eom = the end-of-message flag
  ELSE
    dataLen = the datagram maximum ;  eom = 0
  header.length   = (header size + dataLen) BITOR the data flag BITOR eom
  header.sequence = sock.sendSequence, then increment it
  copy dataLen bytes of the message after the header
  sock.canSend = false                       # no further reliable message until
                                             # this one is acknowledged
  IF the transport write failed  RETURN -1
  sock.lastSendTime = now ;  RETURN 1
```

**Invariants** — **the whole message is retained**, not just the unsent remainder, because retransmission needs the
current fragment and the shift on acknowledgement needs the rest. The store is 8192 bytes per direction per
connection ([`net.h`](net.h.md)).

**The sequence number counts fragments, not messages.** Each fragment gets the next number, which is what lets an
acknowledgement identify a fragment.

**The connection is closed to further reliable messages until this one completes**, which is the interlock the
caller must respect ([`net.h`](net.h.md#net_sendmessage-net_sendunreliablemessage)) and the reason the server
buffers reliable data across frames.

## `SendMessageNext`, `ReSendMessage`

**Contract** — send the next fragment of the outstanding message with a fresh sequence number, clearing the pending
flag; and resend the current fragment with the *same* sequence number it already had, counting the retransmission.

**Invariants** — retransmission reuses the sequence number, so the receiver's duplicate detection works. The
distinction between advancing and resending is the whole of the protocol's reliability.

## `Datagram_SendUnreliableMessage`

**Contract** — sends a message in one datagram with the unreliable flag and its own sequence number. A message too
large for one datagram is a fatal error.

**Invariants** — **no fragmentation and no retransmission.** An unreliable message must fit, which is why the
server's frame datagram is bounded at 1024 bytes
([`quakedef.h`](quakedef.h.md)) and why an overflowing frame is dropped whole
([`common.c`](common.c.md#sz_getspace)).

## `Datagram_GetMessage`

**Contract** — reads and processes every pending datagram for a connection. Retransmits the outstanding fragment
first if a second has passed since it was sent. Returns 1 when a complete reliable message is in the incoming
buffer, 2 when an unreliable one is, 0 when nothing arrived, and −1 on a transport error.

```text
FUNCTION datagram_get_message(sock) -> int
  # Retransmit before reading, if the outstanding fragment has gone unanswered.
  IF NOT sock.canSend AND more than 1 second since the last send
    re_send_message(sock)

  LOOP
    length = read a datagram
    IF nothing arrived  BREAK
    IF the read failed  print "Read error" ;  RETURN -1
    IF the sender's address is not this connection's  CONTINUE    # forged
    IF length < the header size  count a short packet ;  CONTINUE

    length, flags = the header's first word, split
    IF flags HAS the control flag  CONTINUE          # a connection packet;
                                                     # not ours
    sequence = the header's second word

    IF flags HAS the unreliable flag
      IF sequence < the expected one  print "stale datagram" ;  RETURN 0
      IF sequence != the expected one
        count (sequence - expected) DROPPED datagrams and report
      the expected one = sequence + 1
      copy the payload INTO the incoming buffer ;  RETURN 2

    IF flags HAS the acknowledge flag
      IF sequence != sock.sendSequence - 1  print "Stale ACK" ;  CONTINUE
      IF sequence != sock.ackSequence       print "Duplicate ACK" ;  CONTINUE
      sock.ackSequence = sock.ackSequence + 1
      # Shift the acknowledged fragment out of the retained message.
      sock.sendMessageLength = sock.sendMessageLength - the datagram maximum
      IF anything remains
        move the remainder to the front ;  sock.sendNext = true
      ELSE
        sock.sendMessageLength = 0 ;  sock.canSend = true    # the message is
                                                             # complete
      CONTINUE

    IF flags HAS the data flag
      # Acknowledge it FIRST, unconditionally — even a duplicate.
      send an acknowledgement carrying this sequence number
      IF sequence != the expected receive sequence
        count a duplicate ;  CONTINUE                # out of order or repeated
      the expected receive sequence = sequence + 1
      append the payload TO the reassembly buffer
      IF flags HAS the end-of-message flag
        copy the reassembly buffer INTO the incoming buffer
        reset the reassembly buffer ;  RETURN 1
      CONTINUE
```

**Invariants** — eight decisions, and every one is part of the wire behaviour.

**Retransmission is timer-driven at one second**, checked on read rather than on a separate timer. So a lost
fragment costs a full second of latency — which is acceptable on a local network and is why this protocol is
unusable across the internet.

**A datagram from an unexpected address is discarded silently.** That is the only anti-spoofing measure, and it is
address-based.

**A control-flagged packet is ignored here**, because it belongs to the connection handshake, which runs on a
separate socket.

**Unreliable datagrams are sequence-checked and gaps are counted but not recovered.** A stale one is dropped and a
gap is reported; the engine's statistics come from this counting.

**An acknowledgement is validated twice** — against the last sent sequence and against the acknowledge counter — so
a duplicate or stale acknowledgement cannot advance the stream.

**Acknowledging a data fragment happens before the duplicate check**, so a retransmitted fragment is acknowledged
again. That is required: the original acknowledgement may have been the packet that was lost.

**A fragment out of order is discarded, not buffered.** So reassembly is strictly in order and a reordered network
degrades to retransmission.

**The flag tests are in a fixed order** — unreliable, acknowledge, data — and the flags are mutually exclusive
except that end-of-message accompanies data.

**Notes** — a disabled line drops one datagram in seven at random. That is a packet-loss simulator, and it is worth
reproducing in a rebuild: this protocol's behaviour under loss is very hard to reason about and easy to test this
way.

## `Datagram_CanSendMessage`, `Datagram_CanSendUnreliableMessage`

**Contract** — report whether a reliable message would be accepted — sending the pending next fragment first if
one is due — and that an unreliable one always would.

**Invariants** — the side effect of sending a pending fragment from a *query* is deliberate: the server calls the
query every frame ([`sv_main.c`](sv_main.c.md#sv_sendclientmessages)), which is what drives fragmentation forward
without a separate pump.

## `Datagram_Close`, `Datagram_Shutdown`, `Datagram_Init`, `Datagram_Listen`

**Contract** — close a connection through the transport; shut every transport down; initialize every available
transport and register it; and start or stop listening on each.

**Invariants** — this driver holds a table of transport drivers and loops over all of them
([`net.h`](net.h.md#the-two-driver-records)), which is how one server listens on several transports at once.

## `Datagram_SearchForHosts`, `_Datagram_SearchForHosts`

**Contract** — broadcast a server-information request on every transport and collect the replies into the host
cache, deduplicating by address and resolving a reply's address from where it came.

**Invariants** — discovery is a **broadcast request and a unicast reply**, and the reply carries only a port
([`net.h`](net.h.md#the-connection-protocol)) — the address being wherever the reply came from. That is the
multi-homed-host solution the header's comment describes.

## `Datagram_Connect`, `_Datagram_Connect`

**Contract** — sends a connection request to a named host, retrying a few times, and on acceptance opens a new
socket to the port the reply named. Handles a rejection by reporting its reason.

**Invariants** — **the accepted connection moves to a new port**, so the server's control socket stays free for
further requests. The client learns that port from the acceptance reply.

## `Datagram_CheckNewConnections`, `_Datagram_CheckNewConnections`

**Contract** — reads the control socket for connection requests and information queries; answers the queries
directly; and for a request, allocates a connection, opens a socket for it, and replies with that socket's port.
Re-sends the acceptance to a repeated request from an address already connected.

**Invariants** — **a repeated request from a connected address is answered again** rather than creating a second
connection, because the acceptance reply may have been lost. Without that, a lost acceptance leaves the client
retrying forever against a server that thinks it is already connected.

The information queries — server details, a player's details, a rule's value — are answered here, on the control
socket, without a connection. That is how the server browser works.
