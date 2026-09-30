# WinQuake/net_bsd.c

> The Unix driver table: names which message drivers and which transport drivers this build contains, in the order they are tried.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net_loop.h`](net_loop.h.md) · [`net_dgrm.h`](net_dgrm.h.md) · [`net_udp.h`](net_udp.h.md)
**Used by** — [`net_main.c`](net_main.c.md) reads both tables
**Tier floor** — none

## Purpose

Configuration expressed as a pair of tables. One entry per driver, each entry a name plus the function pointers that
make up the interface in [`net.h`](net.h.md). There is one such file per platform — this one, [`net_win.c`](net_win.c.md)
and [`net_dos.c`](net_dos.c.md) — and they differ only in which drivers are listed. Swapping the file is how the build
selects transports.

## State

```text
VARIABLE net_drivers    = [ loopback driver, datagram driver ]
VARIABLE net_landrivers = [ UDP transport ]
```

**Invariants** — **order is the priority order.** The loopback driver is first, so a connection to the local server
resolves before any socket is tried ([`net_main.c`](net_main.c.md#net_connect)). A rebuild must keep the loopback
first.

Each entry begins with the driver **disabled**; initialization enables it if it succeeds, and the dispatcher skips
disabled entries. So a missing transport is not an error, it is one fewer table row in play.

**Notes** — a rebuild replaces these tables with whatever its language uses for a list of implementations of an
interface. The load-bearing content is only: *which* transports exist on this platform, and *in what order*. On Unix
that is loopback, then datagram-over-UDP, and nothing else.
