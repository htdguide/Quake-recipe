# WinQuake/net_dos.c

> The DOS driver table: loopback, serial and datagram, with IPX and two vendor Ethernet stacks as transports, each row compiled in only if its library is present.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net_loop.h`](net_loop.h.md) · [`net_dgrm.h`](net_dgrm.h.md) · [`net_ser.h`](net_ser.h.md) · [`net_ipx.h`](net_ipx.h.md) · [`net_bw.h`](net_bw.h.md)
**Used by** — [`net_main.c`](net_main.c.md) reads both tables
**Tier floor** — none

## Purpose

The DOS variant of the table described in [`net_bsd.c`](net_bsd.c.md). Its interest is historical: it shows that the
engine's network abstraction was built to accommodate transports with no shared heritage — real-mode IPX through
interrupt 0x7A, and a vendor TCP stack reached through a device ioctl.

## State

```text
VARIABLE net_drivers    = [ loopback, serial, datagram ]
VARIABLE net_landrivers = [ IPX, Beame & Whiteside TCP ]   # rows present per build flags
```

**Notes** — nothing here is load-bearing for a rebuild beyond the confirmation that the transport interface in
[`net.h`](net.h.md) is genuinely portable: it was filled three times over, by families that share no address format,
no error codes and no blocking model.
