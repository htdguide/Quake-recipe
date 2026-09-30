# QW/docs/glqwcl-readme.txt

> Data: the hardware client's release note — the driver requirement, the switches worth changing, and what differs from the software client.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

The QuakeWorld counterpart of [`glqnotes.txt`](../../WinQuake/glqnotes.txt.md) — read that twin for the substance, which is the same:
the driver requirement, the switch-per-symptom troubleshooting, and the fact that the software renderer remains the fallback.

## State

Data; no run-time state.

## What it adds

**That the two renderers are the same client** with the same networking, so nothing about the connection behaves differently. The only
observable differences are visual: round particles, filtered textures, the round dynamic-light blobs
([`gl_rlight.c`](../client/gl_rlight.c.md)), and no disc indicator during a load
([`gl_draw.c`](../client/gl_draw.c.md)).

**That the frame-rate limit matters more here**, because the hardware renderer can exceed it easily and the frame rate is the packet
rate ([`cl_main.c`](../client/cl_main.c.md)). A player who removes the limit floods their own uplink — which is the clearest
user-visible consequence of the packet-per-frame design and the reason the limiter is clamped rather than advisory.

**Notes** — the last point is the one worth carrying into a rebuild: **if packets are tied to frames, the frame limiter is part of the
network configuration and must be documented as such.**
