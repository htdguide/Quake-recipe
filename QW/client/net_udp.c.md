# QW/client/net_udp.c

> The datagram transport for Unix.

**Needs** — as [`net_udp.c`](../../WinQuake/net_udp.c.md)
**Used by** — as [`net_udp.c`](../../WinQuake/net_udp.c.md)
**Tier floor** — as [`net_udp.c`](../../WinQuake/net_udp.c.md)

## Purpose

Read [`net_udp.c`](../../WinQuake/net_udp.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`net_udp.c`](../../WinQuake/net_udp.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Rewritten against the concrete address record ([`net.h`](net.h.md)) instead of the opaque blob, and reduced to what one program needs: open a socket, read, write, broadcast, and convert addresses. The non-blocking requirement and the treat-connection-refused-as-no-data rule are unchanged and remain essential. The partial-address completion is gone — a server is named in full or by a directory listing.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
