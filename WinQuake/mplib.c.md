# WinQuake/mplib.c

> Thin wrappers around a third-party matchmaking library's queue primitives, each one a far call with the arguments marshalled.

**Needs** — [`mpdosock.h`](mpdosock.h.md) · [Seam: Unreliable datagram transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`mplpc.c`](mplpc.c.md)
**Tier floor** — T1: the calls cross a memory-model boundary the language must express

## Purpose

Glue, not design. Each function is one call into a resident library through a pointer the loader supplied, with the
arguments placed where that library expects them. It exists because the library was written for a different memory
model than the engine.

## State

```text
VARIABLE the library's entry-point table, supplied at load
RECORD RTQ_NODE          # a queue node owned by the library
```

## The operations

**Contract** — yield to the library; post a notification to the host's event queue; wait for the library; read a
queue's counter; move a node between queues; take a node from a queue; take the master node and its size; flush a
range of nodes from one queue to another; read a 64-bit counter as two halves; run the library's consistency check;
wake the library.

**Invariants** — **nodes are owned by the library and only moved between its queues**, never allocated or freed here.
The engine's side of the relationship is: take a node, read it, give it back.

**Notes** — nothing here is load-bearing for a rebuild. It is recorded so the reader of
[`mplpc.c`](mplpc.c.md) and [`net_mp.c`](net_mp.c.md) knows where the far calls went and does not go looking for a
protocol that is not there. The useful line is the one in [`net_mp.c`](net_mp.c.md): a hosted service entered the
engine as a transport driver.
