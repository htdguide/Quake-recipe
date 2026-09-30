# WinQuake/net_udp.c

> The canonical transport driver: non-blocking datagram sockets, one control socket plus one accept socket, and the partial-address rule that lets a player type the last octet of a server on their own network.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net.h`](net.h.md) · [`net_udp.h`](net_udp.h.md) · [Seam: Unreliable datagram transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`net_bsd.c`](net_bsd.c.md) lists it as a transport; [`net_dgrm.c`](net_dgrm.c.md) calls every operation through the table
**Tier floor** — none

## Purpose

Fills the transport half of the network interface ([`net.h`](net.h.md)) with datagram sockets. This is the one
transport worth rebuilding, and it is the reference for the other five
([`net_wins.c`](net_wins.c.md), [`net_wipx.c`](net_wipx.c.md), [`net_ipx.c`](net_ipx.c.md),
[`net_bw.c`](net_bw.c.md), [`net_mp.c`](net_mp.c.md)), which implement the same nineteen operations over other
families.

The interface it fills is deliberately thin: open and close a socket, read, write, broadcast, and convert between an
address and its printable form. Everything above it — sequencing, reliability, fragmentation, timeouts — lives in
[`net_dgrm.c`](net_dgrm.c.md), so a new transport is a day's work and never touches the protocol.

## State

```text
VARIABLE net_acceptsocket    : handle = none    # bound to the game port, server only
VARIABLE net_controlsocket   : handle           # bound to an arbitrary port, always open
VARIABLE net_broadcastsocket : handle = none    # whichever socket broadcast was enabled on
VARIABLE broadcastaddr       : address          # the broadcast address at the game port
VARIABLE myAddr              : address bits     # this machine's own address
```

**Invariants** — the **control socket** and the **accept socket** are distinct and serve different roles. The control
socket is open for the whole session on an arbitrary port and carries connection requests and discovery; the accept
socket is bound to the well-known game port and exists only while listening. A client therefore never binds the game
port, so two clients can run on one machine — and a connection, once established, moves to yet another port
([`net_dgrm.c`](net_dgrm.c.md#datagram_connect-_datagram_connect)). A rebuild that collapses these into one socket breaks both
properties.

**At most one socket may be broadcast-enabled**, and attempting a second is a fatal error. This is a real constraint
of the era's stacks, not of the design; a rebuild may lift it.

## `UDP_Init`

**Contract** — refuses if disabled on the command line. Resolves this machine's own address, defaults the server's
advertised name to the machine name if the player never set one, opens the control socket, composes the broadcast
address at the game port, and records the printable local address with the port stripped. Returns the control socket
or failure.

**Invariants** — this machine's own address is needed **before** any connection, because the partial-address rule
below completes a typed address from it.

## `UDP_Listen`

**Contract** — takes a flag; opens the accept socket on the game port, or closes it. Idempotent in both directions.
Failure to bind the game port is fatal.

**Invariants** — binding the game port is what makes this process *the* server on this machine, so the failure is
fatal rather than a degraded mode: two servers on one port would each see half the traffic.

## `UDP_OpenSocket`, `UDP_CloseSocket`

**Contract** — create a datagram socket, switch it to **non-blocking**, bind it to the given port on all local
addresses, and return it; or close one, clearing the broadcast record if it was the broadcast socket. Port zero means
any free port.

**Invariants** — **non-blocking is not optional.** The whole engine is one thread with one frame loop
([`host.c`](host.c.md#host_frame)); a read that blocked would stall rendering. Every read therefore reports
*would-block* as "no data", which the layer above treats as an empty poll.

## `PartialIPAddress`

**Contract** — takes a partial dotted address, possibly with a port suffix; fills in the missing leading octets from
this machine's own address and returns the completed address. Rejects malformed input.

```text
FUNCTION partial_address(text) -> optional<address>
  prefix text with a separator so a leading separator is not special
  addr = 0 ;  mask = all ones
  WHILE the next character is a separator
    read up to 3 digits as num                  # more than 3 is malformed
    reject if the character after them is neither a separator, a port mark, nor the end
    reject if num > 255
    mask = mask SHIFTED LEFT one octet          # one fewer octet taken from myAddr
    addr = (addr SHIFTED LEFT one octet) + num
  port = the suffix if present, else the game port
  RETURN (myAddr AND mask) OR addr              # both in network order
```

**Invariants** — **the mask is built by counting the octets actually typed**, shifting left once per octet, so the
untyped high octets come from this machine's address and the typed low octets from the text. Typing `12` on
`192.168.0.x` reaches `192.168.0.12`. This is load-bearing: it is the interface a 1996 LAN party used, and it is
still the shortest way to join a server on the same network.

Note the mask starts as *all ones* and shifts left, so a fully typed four-octet address masks `myAddr` away
completely and the rule degrades correctly to an absolute address.

## `UDP_Connect`

**Contract** — does nothing and reports success.

**Invariants** — the transport is connectionless; every send carries its destination. The operation exists only
because the interface has a slot for it, and transports that *are* connection-oriented fill it.

## `UDP_CheckNewConnections`

**Contract** — reports the accept socket when data is waiting on it, otherwise nothing. Does not read.

**Invariants** — it **peeks** rather than reads, because the layer above wants to read the message itself through the
ordinary path. A rebuild without a byte-count query can poll readability instead.

## `UDP_Read`, `UDP_Write`

**Contract** — read one datagram into a buffer, reporting the sender's address; or send one to an address. Both
report zero bytes rather than an error when the operation would block. A read also treats *connection refused* as no
data.

**Invariants** — **treating connection-refused as no data** is what keeps the engine alive when a peer vanishes: on
these stacks an unreachable port surfaces as an error on the *next read* of an unrelated datagram, and failing there
would kill a working session. A rebuild on a stack with the same behaviour needs the same suppression.

## `UDP_Broadcast`, `UDP_MakeSocketBroadcastCapable`

**Contract** — enable broadcast on the socket if it is not already the broadcast socket, then send to the broadcast
address at the game port. Fatal error if a different socket already holds broadcast rights.

**Invariants** — enabling happens lazily on first broadcast, so a client that never opens the server browser never
asks for the right.

## `UDP_AddrToString`, `UDP_StringToAddr`

**Contract** — render an address as four decimal octets and a decimal port separated by a mark, and parse that form
back. The renderer returns a shared buffer, so a caller needing two addresses at once must copy.

**Invariants** — this text form is **player-visible and stored** — it appears in the server browser, in status output
and in the host cache — so it is part of the user interface, not an internal detail.

## `UDP_GetSocketAddr`, `UDP_GetNameFromAddr`, `UDP_GetAddrFromName`, `UDP_AddrCompare`, `UDP_GetSocketPort`, `UDP_SetSocketPort`

**Contract** — report a socket's own bound address, substituting this machine's address for a wildcard; look a name up
from an address and an address up from a name, the latter accepting the partial form above; compare two addresses
reporting equal, or equal-but-for-the-port, or unequal; and read or replace the port of an address.

**Invariants** — the three-way comparison is load-bearing. [`net_dgrm.c`](net_dgrm.c.md) uses *equal-but-for-the-port*
to recognize that a reply came from the same host on the new port the handshake moved to, and *fully equal* to reject
duplicates. Collapsing it to a boolean breaks the handshake.

**Notes** — reporting this machine's address in place of a wildcard bind matters because the wildcard is what gets
sent to a peer otherwise, and a peer cannot reply to it.
