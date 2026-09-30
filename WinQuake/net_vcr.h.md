# WinQuake/net_vcr.h

> Declares one message-layer driver's eleven operations: the recording and replay message layer.

**Needs** — [`net.h`](net.h.md)
**Used by** — [`net_main.c`](net_main.c.md), which fills a driver record from these; implemented by [`net_vcr.c`](net_vcr.c.md)
**Tier floor** — none

## Purpose

One of the **message-layer** drivers of [`net.h`](net.h.md) — the layer that provides ordering,
reliability and fragmentation, not the layer that moves bytes. This file is only its function
declarations; the contracts belong to [`net.h`](net.h.md#the-two-driver-records) and the
behaviour to [`net_vcr.c`](net_vcr.c.md).

## State

Stateless.

## Contract

**Contract** — the eleven operations of the message-driver record, named with the `VCR` prefix:
initialize, shut down, start or stop listening, search for hosts, connect, check for a new inbound
connection, get a message, send a reliable message, send an unreliable message, ask whether either kind
can be sent now, and close a connection. Their semantics are given once in
[`net.h`](net.h.md).

**Notes** — the engine has room for eight of these and loops over every one that initialized
successfully, which is how a server listens on several transports at once
([`net.h`](net.h.md#the-two-driver-records)). A rebuild with one transport plus loopback needs two.
