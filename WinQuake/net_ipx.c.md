# WinQuake/net_ipx.c

> The DOS IPX transport: the datagram interface driven by real-mode interrupt calls, with receives completing asynchronously into a ring of pre-posted buffers.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`net.h`](net.h.md) · [`net_ipx.h`](net_ipx.h.md) · [`dosisms.h`](dosisms.h.md) · [Seam: Unreliable datagram transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`net_dos.c`](net_dos.c.md) lists it as a transport
**Tier floor** — T1: needs a fixed low-memory address for buffers the transport writes to asynchronously

## Purpose

The same interface as [`net_udp.c`](net_udp.c.md), but it is the one transport whose *shape* differs rather than just
its address format, and that difference is instructive.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs from the reference driver

**Receives are posted, not polled.** A request block is handed to the transport with a buffer, and the transport fills
it and marks it complete at some later moment, possibly during an interrupt. So the driver keeps a **ring of posted
request blocks** and, on each read, walks the ring taking every block the transport has marked done, moving it to a
ready list, and re-posting it.

```text
FUNCTION process_ready_list(socket)
  WHILE the oldest posted block is marked complete
    move it to the ready list
    post a fresh block in its place          # the ring must never run empty
FUNCTION read(socket, buf) -> int
  process_ready_list(socket)
  IF the ready list is empty  RETURN 0
  take the oldest ready entry ;  copy out its bytes and sender address
  return it to the posted ring
```

**Invariants** — **a socket with no posted block loses datagrams silently**, so re-posting happens before the data is
consumed, not after. This is the general hazard of a completion-based transport, and a rebuild on one (overlapped
reads, io_uring, a callback API) inherits it verbatim.

**The buffers must live at a fixed low address** the real-mode transport can reach, which is the tier floor above and
the only genuinely T1 requirement in the whole network chapter.

**A yield call is made while waiting**, letting the transport's own interrupt handler run — the same need
[`net_wins.c`](net_wins.c.md) meets with a blocking hook.

**It registers a periodic procedure** with the timer queue in [`net_main.c`](net_main.c.md#net_poll-schedulepollprocedure) so the ring is
serviced even in a frame that never reads, which is how a listening server keeps its buffers posted.
