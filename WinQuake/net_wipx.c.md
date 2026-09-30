# WinQuake/net_wipx.c

> The Windows IPX transport: the datagram interface over a network/node/socket address family, proving the protocol layer is address-family-agnostic.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net.h`](net.h.md) · [`net_wipx.h`](net_wipx.h.md) · [`winquake.h`](winquake.h.md) · [Seam: Unreliable datagram transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`net_win.c`](net_win.c.md) lists it as a transport
**Tier floor** — none

## Purpose

The same nineteen operations as [`net_udp.c`](net_udp.c.md) over IPX. Present in the recipe for one reason: it is the
evidence that nothing above the transport knows what an address is.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs from the reference driver

**An address is a twelve-byte network-plus-node-plus-socket triple**, not four octets, and its printable form is
hexadecimal. So the opaque sixteen-byte address blob in [`net.h`](net.h.md) is not over-engineering — it is sized for
this family, and everything above it treats an address as bytes it may copy, compare and print but never interpret.
A rebuild should keep that opacity even if it only ever ships one family.

**Sockets are opened from a small fixed pool**, each with its own receive buffer, because the family's send and
receive calls need a caller-supplied buffer that outlives the call. The layer above sees ordinary handles.

**There is no partial-address completion**: an IPX address has no hierarchy to complete from, so the rule in
[`net_udp.c`](net_udp.c.md#partialipaddress) has no counterpart and a player must broadcast to find a server.

**Broadcast is the normal case** on this family rather than a right that must be requested.
