# QW/qwfwd/qwfwd.c

> A datagram forwarder: a tiny program launched per connection by the system's service dispatcher that relays packets between one client and one server, so a server behind a restricted network can still be reached.

**Needs** — [`misc.c`](misc.c.md) · [Seam: Unreliable datagram transport](../../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport) · [Seam: Operating system services](../../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — nothing in the engine; it is a separate program
**Tier floor** — T3: it is a hundred lines of socket relay with no timing requirement of its own

## Purpose

An auxiliary program, not part of the engine. It exists because a game server sometimes cannot be reached directly — behind an address
translator, or on a network that only permits certain hosts — and the channel's design makes a relay unusually easy: **every packet is
self-contained and the protocol has no notion of the address it is talking to beyond the channel's own record**
([`net_chan.c`](../client/net_chan.c.md)).

It is in the recipe for one reason: it is the proof that the protocol is relayable, which a rebuild wanting a modern relay, a
matchmaking service or a NAT traversal scheme needs to know.

## State

```text
VARIABLE the destination address and port, from the command line
VARIABLE the socket toward the server
VARIABLE the client's address, learned from the first packet
```

## `main`, `connectsock`, `NET_Init`

**Contract** — launched with the destination host and port, with the client's socket already connected by the service dispatcher.
Opens a socket toward the destination, then loops: read from either side and write to the other, until a period of silence ends the
program.

```text
FUNCTION main(dest_host, dest_port)
  the inbound socket is inherited, already bound to the client
  outbound = a datagram socket connected to (dest_host, dest_port)
  LOOP
    wait for either socket to be readable, with a timeout
    IF the timeout expired  EXIT
    IF the inbound socket has a packet   forward it to outbound
    IF the outbound socket has a packet  forward it to inbound
```

**Invariants** —

- **One process per client, launched on demand and exiting on silence.** There is no connection table, no state, no configuration
  beyond the destination. The system's service dispatcher provides the per-client process and the inherited socket, which is why this
  is a hundred lines.
- **The relay is transparent to the protocol.** The server sees the relay's address as the client's, so the client's connection
  identifier ([`net_chan.c`](../client/net_chan.c.md)) is what actually distinguishes players — a relay would otherwise merge them. That
  identifier was introduced for address translators
  ([`net_chan.c`](../client/net_chan.c.md)) and it is what makes relaying work as well.
- **The silence timeout is the only lifetime management**, which is correct for a connectionless relay and means a crashed client
  costs one idle process for the timeout.
- Packets are forwarded **unexamined**, so the relay needs no protocol version and survives protocol changes.

**Notes** — the transferable observation: **a protocol whose packets are self-contained and whose sessions are identified by a value
inside the packet rather than by the address can be relayed by a program this small.** That is a property worth designing for, and it
is the argument for the connection identifier being part of the header rather than inferred.

The program is adapted from a general-purpose port redirector, credited in the source.
