# WinQuake/net_win.c

> The Windows driver table: loopback, serial and datagram as message drivers, with UDP, IPX and MPath as the datagram transports.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net_loop.h`](net_loop.h.md) · [`net_dgrm.h`](net_dgrm.h.md) · [`net_ser.h`](net_ser.h.md) · [`net_wins.h`](net_wins.h.md) · [`net_wipx.h`](net_wipx.h.md) · [`net_mp.h`](net_mp.h.md)
**Used by** — [`net_main.c`](net_main.c.md) reads both tables
**Tier floor** — none

## Purpose

The richest of the three driver tables, because Windows was the shipping platform. See
[`net_bsd.c`](net_bsd.c.md) for what a driver table is and why order matters.

## State

```text
VARIABLE net_drivers    = [ loopback, serial (modem/null-modem), datagram ]
VARIABLE net_landrivers = [ Winsock UDP, Winsock IPX, MPath ]
```

**Invariants** — the serial driver sits **above** the datagram driver in the message-driver table but is a *peer* of
it, not a transport: it implements the whole reliable-message interface itself
([`net_ser.c`](net_ser.c.md)) because a serial line is a byte stream, not a datagram channel. That asymmetry is the
reason the interface is split into two levels at all.

The three transports are tried in table order for each connect attempt, so a bare host name reaches UDP first.

**Notes** — only the loopback and datagram-over-UDP rows are worth reproducing. Serial, IPX and MPath are dead
transports; their rows document that the datagram layer was written against three different address families, which
is why an address is an opaque sixteen-byte blob in [`net.h`](net.h.md) rather than an IP address.
