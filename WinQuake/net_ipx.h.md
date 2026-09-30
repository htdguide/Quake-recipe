# WinQuake/net_ipx.h

> Declares one transport driver's fifteen operations: DOS IPX.

**Needs** — [`net.h`](net.h.md)
**Used by** — [`net_main.c`](net_main.c.md) fills a transport-driver record from these; implemented by [`net_ipx.c`](net_ipx.c.md)
**Tier floor** — none; this is the seam

## Purpose

One implementation of [Seam: Unreliable datagram
transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport), declared. The file carries
only names; the contracts are stated once in [`net.h`](net.h.md#the-two-driver-records) and the
behaviour in [`net_ipx.c`](net_ipx.c.md).

## State

Stateless.

## Contract

**Contract** — the fifteen operations of the transport-driver record, named with the `IPX` prefix:
open and close a socket, connect, check for new connections, read, write, broadcast, convert an address
to and from text, read a socket's own address, resolve a name both ways, compare two addresses, and read
or replace the port within one.

**Invariants** — the address is opaque to the engine throughout
([`net.h`](net.h.md)), so this driver is the only code that knows what is inside one. That opacity is
what lets a byte-stream transport and a datagram transport share the interface.

**Notes** — six transports ship in this tree with identical interfaces and wildly different
implementations. A rebuild needs one.
