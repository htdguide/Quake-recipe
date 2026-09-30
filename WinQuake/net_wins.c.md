# WinQuake/net_wins.c

> The Windows datagram transport: the same nineteen operations as the reference UDP driver, reached through function pointers loaded from the socket library at run time.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net.h`](net.h.md) · [`net_wins.h`](net_wins.h.md) · [`winquake.h`](winquake.h.md) · [Seam: Unreliable datagram transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`net_win.c`](net_win.c.md) lists it as a transport
**Tier floor** — none

## Purpose

Same contract, same semantics, same partial-address rule as [`net_udp.c`](net_udp.c.md) — read that twin for the
operations. Only three things differ, and only they are worth recording.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs from the reference driver

**Every socket call is an indirect call through a pointer resolved at startup.** The library is loaded by name and
each entry point looked up, so the engine runs on a machine with no network stack installed instead of failing to
start. That is the load-bearing decision: **a missing transport must degrade to one fewer driver, never to a launch
failure** — the same principle as the disabled-row convention in [`net_bsd.c`](net_bsd.c.md).

**The stack is explicitly started and stopped**, and the version it reports is checked against a floor. A rebuild on a
platform needing the same ceremony does it here.

**A blocking hook is installed** that pumps the window message queue while a socket call blocks. Name resolution is
the only blocking call left, and without the hook the window would freeze during it. In a rebuild this is the
platform's "do not block the UI thread" problem; solve it with whatever the platform offers — the decision is only
that *name resolution is allowed to block and nothing else is*.

**It also enumerates the local addresses** rather than trusting a single resolved name, because a multi-homed machine
resolves its own name to an address a LAN peer cannot reach.

**Notes** — the run-time lookup also means the driver reports which stack it found, which is the only diagnostic a
player had when a game would not see the network.
