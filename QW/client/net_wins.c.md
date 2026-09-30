# QW/client/net_wins.c

> The datagram transport for Windows.

**Needs** — as [`net_wins.c`](../../WinQuake/net_wins.c.md)
**Used by** — as [`net_wins.c`](../../WinQuake/net_wins.c.md)
**Tier floor** — as [`net_wins.c`](../../WinQuake/net_wins.c.md)

## Purpose

Read [`net_wins.c`](../../WinQuake/net_wins.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`net_wins.c`](../../WinQuake/net_wins.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The same reduction as [`net_udp.c`](net_udp.c.md). The run-time resolution of the socket library's entry points and the degrade-rather-than-fail rule are unchanged; the blocking hook around name resolution is unchanged and still necessary.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
