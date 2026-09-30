# WinQuake/mplpc.c

> A sockets-shaped facade over a third-party matchmaking library: each familiar socket call becomes a request placed in a shared queue, with strings and buffers copied across a memory-model boundary.

**Needs** — [`mpdosock.h`](mpdosock.h.md) · [`mplib.c`](mplib.c.md) · [Seam: Unreliable datagram transport](../SYSTEM-REQUIREMENTS.md#seam-unreliable-datagram-transport)
**Used by** — [`net_mp.c`](net_mp.c.md) calls it believing it is a socket library
**Tier floor** — T1: every call copies between two address representations

## Purpose

The reason [`net_mp.c`](net_mp.c.md) can be a near-copy of [`net_udp.c`](net_udp.c.md): this file *is* the socket API,
reimplemented over somebody else's service. It answers a question a rebuilder will have — how did a commercial
matchmaking network get into a 1996 engine without touching the engine? — and the answer is a shim at the narrowest
possible place.

## State

```text
VARIABLE socket map               # our handles to the service's
VARIABLE the shared request/response queues (see mplib.c)
VARIABLE a cached host entry for name lookups
```

## The copy helpers

**Contract** — copy a block or a string, bounded or not, *into* the library's address space or *out of* it.

**Invariants** — **nothing is shared by reference across the boundary; everything is copied.** That is the whole reason
these exist, and in a rebuild the equivalent is whatever marshalling the chosen transport library demands. The problem
being solved is: the caller's buffer and the service's buffer are not the same kind of thing.

## The socket facade

**Contract** — bind, close, send, receive, address queries, name lookup and the socket options the caller uses —
each shaped exactly like the platform call of the same name, each implemented as: allocate a request node from the
shared queue, fill it with the marshalled arguments, hand it to the library, wait for the reply node, unmarshal the
result, free the node.

```text
FUNCTION facade_call(op, args) -> result
  node = take a free node from the request queue, sized for args
  marshal args INTO the node
  hand the node to the library ;  wake it
  wait for the matching reply node
  unmarshal the result ;  return the node to the free queue
  RETURN the result, translated to this platform's error codes
```

**Invariants** — a node must be returned to the free queue on **every** path, including the error paths, because the
queue is fixed-size and a leaked node is a permanently smaller queue. The failure is gradual, which is why it is worth
naming.

**Notes** — the load-bearing observation for a rebuild is only the placement of the seam: if the transport can be made
to look like the six-call interface in [`net.h`](net.h.md), or failing that like the platform's socket calls, then a
whole alternative network can be added without the protocol layer learning of it. That is an argument for keeping the
transport interface as thin as the original made it.
