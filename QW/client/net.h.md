# QW/client/net.h

> The network interface, rewritten: an address that is bytes and a port, a socket the program owns directly, and the sequenced channel's record.

**Needs** — nothing
**Used by** — [`net_chan.c`](net_chan.c.md) · [`net_udp.c`](net_udp.c.md) · [`net_wins.c`](net_wins.c.md) · [`cl_main.c`](cl_main.c.md) · [`sv_main.c`](../server/sv_main.c.md)
**Tier floor** — none

## Purpose

The replacement for [`net.h`](../../WinQuake/net.h.md), and the differences say what the original's abstraction was costing.

## State

```text
RECORD NetAddr
  ip   : 4 bytes
  port : 2 bytes, in network order
  pad  : 2 bytes

CONSTANT max_latent                     # the sent-packet log's size
RECORD Netchan                          # see net_chan.c for the whole record
  the sequence counters and the reliable alternating bits
  the composing buffer and the in-flight buffer
  the rate and the virtual clock
  the sent-packet log, and the smoothed statistics
FUNCTION Netchan_Init, Setup, Transmit, Process, CanPacket, CanReliable,
         OutOfBand, OutOfBandPrint
FUNCTION NET_Init, Shutdown, GetPacket, SendPacket,
         CompareAdr, CompareBaseAdr, AdrToString, StringToAdr, IsClientLegal
VARIABLE net_from, net_message
```

## What differs from the original

**There is no driver table.** The original's two-level abstraction — message drivers over transport drivers, each a table of
function pointers ([`net.h`](../../WinQuake/net.h.md)) — is gone. There is one socket and one set of functions to use it. The
original's abstraction existed to support serial links, three address families and a loopback driver; with a single program talking
over one family, all of it is dead weight.

**An address is a concrete record**, not an opaque sixteen-byte blob. The original hides the address so that IPX and serial can
share the interface; here there is one family, so the address is what it is, and the code that compares and prints addresses is
straightforward. **Comparing the host part alone is a distinct operation** from comparing the whole address, and both are needed
([`net_chan.c`](net_chan.c.md)'s port-migration rule, [`sv_main.c`](../server/sv_main.c.md)'s client matching).

**There is no loopback driver and no connection pool.** A client and a server are separate programs; a player hosting a game runs
both and they talk over the socket like anyone else. That removes the original's most subtle machinery
([`net_loop.c`](../../WinQuake/net_loop.c.md)) and its cost: local play now goes through a real socket, which is slightly slower and
much simpler, and it means single-player and network play are *exactly* the same code path rather than merely similar.

**The channel record is part of this header**, because the channel is the abstraction now. Everything the original expressed as
driver operations — can I send, send reliably, send unreliably, close — is either a field of the channel or a function over it.

**A permitted-address check is declared** ([`cl_main.c`](cl_main.c.md)), which the original has no need for because it had no way to
be told an address by a third party.

**Invariants** —

- **The port is stored in network order and the address bytes are stored as bytes**, so no byte-order conversion happens anywhere
  except at the socket boundary. That removes a whole class of bug the original's opaque blob invites.
- The sent-packet log's size, the statistics smoothing and the rate live here because both programs read them
  ([`sbar.c`](sbar.c.md), [`sv_ccmds.c`](../server/sv_ccmds.c.md)).

**Notes** — the measurement worth keeping: **the original's network layer is eleven files and this one is three**
([`net_chan.c`](net_chan.c.md), [`net_udp.c`](net_udp.c.md) or [`net_wins.c`](net_wins.c.md)). The difference is entirely the
abstraction that supported transports nobody uses. A rebuild should start here and add abstraction only when a second transport
actually arrives.
