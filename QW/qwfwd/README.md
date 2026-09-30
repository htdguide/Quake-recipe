# QW/qwfwd — the datagram forwarder

> A hundred lines that relay packets between a client and a server, and the proof that the protocol can be relayed at all.

A server behind an address translator or a restricted network cannot always be reached directly. This program, launched per client by
the system's service dispatcher, forwards packets in both directions until the connection goes quiet.

[`qwfwd.c`](qwfwd.c.md) — the relay: two sockets, a readiness wait, and a silence timeout. No connection table, no state, no protocol
knowledge.
[`misc.c`](misc.c.md) — copies of the tokenizer and the information dictionary from
[`../client/common.c`](../client/common.c.md), duplicated because the forwarder does not link the engine.
[`qwfwd.001`](qwfwd.001.md) — an accidentally committed project-file backup.

**Why it is in the recipe.** It records a property of the channel worth designing for deliberately: **because every packet is
self-contained and a session is identified by a value inside the packet rather than by its address**
([`../client/net_chan.c`](../client/net_chan.c.md)'s connection identifier), a relay needs no state and no version. A rebuild that wants
a modern equivalent — a hosted relay, a matchmaking service, address-translation traversal — should check that its own protocol still
has that property, and this program is the test.

**Not twinned**: the editor and project files.
