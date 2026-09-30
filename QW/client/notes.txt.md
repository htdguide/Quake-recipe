# QW/client/notes.txt

> Data: a discarded design sketch — the packet sender and receiver as separate threads woken by a timer, with the command record under a lock.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

An abandoned alternative to the shipped design, and worth a page precisely because it was abandoned.

## State

Data; no run-time state.

## What it records

A sketch in which sending and receiving are **independent threads**: a sender woken by a timer or by input, taking a lock over the
current movement command and transmitting it; a receiver blocked on the socket. A flag suppresses the next timer wake-up when input
already caused a send.

**Invariants** — compare what shipped: **one thread, one packet per frame, no locks**
([`cl_input.c`](cl_input.c.md), [`cl_main.c`](cl_main.c.md)). The threaded design would have decoupled the packet rate from the frame
rate, which sounds better and would have cost a lock on the movement command, a second clock, and the loss of the property that a
packet corresponds to exactly one rendered frame — the property the whole prediction and delta scheme depends on
([`client.h`](client.h.md)'s single ring indexed by packet sequence).

The suppression flag in the sketch is the tell: the design already needed a special case to avoid sending twice, before anything was
built.

**Notes** — the lesson for a rebuilder is worth stating directly. **Tying the packet rate to the frame rate is what makes the client's
whole state a single ring**, which in turn makes prediction, delta decompression, latency measurement and diagnostics fall out of one
structure. Decoupling them is the obvious improvement and it dismantles that. If a rebuild does decouple them — and on modern
hardware, where frame rates are far above any sensible packet rate, it probably should — it must first decide what replaces the ring.
