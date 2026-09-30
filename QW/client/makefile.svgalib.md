# QW/client/makefile.svgalib

> Data: the console build's description, and therefore the object list for the client's software renderer on Unix.

**Needs** — nothing
**Used by** — nothing in the engine; it is the build
**Tier floor** — none

## Purpose

The QuakeWorld client's counterpart to
[`Makefile.linuxi386`](../../WinQuake/Makefile.linuxi386.md) — read that twin for the renderer substitution rule, which is the same.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## What it records

**The client's object list**, which is the authoritative answer to what the client is. Compare
[`makefile`](../server/makefile.md)'s much shorter list: the difference is the renderer, the sound, the input, the interface and the
two-dimensional layer.

**The shared files appear in both lists**: the channel, the transport, the movement model and its collision queries, and the
foundation. That overlap is the QuakeWorld architecture, stated as a build.

**Invariants** — the portable inner loops are selected, not the hand-written ones
([Seam: Vectorized inner loops](../../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)), which is again the evidence that the
assembly is optional.
