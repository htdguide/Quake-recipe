# WinQuake/net_mp.c

> The MPath transport: the reference UDP driver over a commercial matchmaking service's socket library.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net.h`](net.h.md) · [`net_mp.h`](net_mp.h.md) · [`mpdosock.h`](mpdosock.h.md) · [Seam: Unreliable datagram transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`net_win.c`](net_win.c.md) lists it as a transport
**Tier floor** — none

## Purpose

Operation for operation the same as [`net_udp.c`](net_udp.c.md) — including the partial-address rule — with the
socket calls directed at a third-party library instead of the system stack.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs from the reference driver

**The stack is somebody else's service.** Its library is reached through the declarations in
[`mpdosock.h`](mpdosock.h.md) and the shims in [`mplpc.c`](mplpc.c.md); the addresses it hands out are the service's,
not the local network's, and the service does the matchmaking that the broadcast discovery in
[`net_main.c`](net_main.c.md#net_slist_f-slist_send-slist_poll-and-the-three-printing-helpers) does on a LAN.

**Notes** — the useful observation for a rebuild is that a hosted matchmaking service slotted in as *a transport
driver*, with no change above it. If a rebuild wants a modern equivalent — a relay, a NAT-punching service, WebRTC
data channels — this is the seam it goes in at, and this file is the precedent that it fits.
