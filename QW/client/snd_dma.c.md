# QW/client/snd_dma.c

> The mixer's front end: channels, spatialization and the ring buffer.

**Needs** — as [`snd_dma.c`](../../WinQuake/snd_dma.c.md)
**Used by** — as [`snd_dma.c`](../../WinQuake/snd_dma.c.md)
**Tier floor** — as [`snd_dma.c`](../../WinQuake/snd_dma.c.md)

## Purpose

Read [`snd_dma.c`](../../WinQuake/snd_dma.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`snd_dma.c`](../../WinQuake/snd_dma.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Nothing of substance. Attenuation is still computed on the client from the server's coefficient and position ([`sv_send.c`](../server/sv_send.c.md)), which is why a sound stays correct as the player moves through it.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
