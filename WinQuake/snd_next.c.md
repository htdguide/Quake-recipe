# WinQuake/snd_next.c

> A minimal audio backend for one platform: opens a device, reports a position, and closes.

**Needs** — [`sound.h`](sound.h.md)
**Used by** — [`snd_dma.c`](snd_dma.c.md) through [Seam: Audio output](../SYSTEM-REQUIREMENTS.md#seam-audio-output)
**Tier floor** — T2

## Purpose

Fifty lines, and the shortest real implementation of the audio seam in the tree — which makes it the best measure
of how small the seam is: **three functions and a record**.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## Contract

**Contract** — `SNDDMA_Init` opens the platform's audio device, allocates a buffer, records the format and returns
whether it worked; `SNDDMA_GetDMAPos` reports the consumed position; `SNDDMA_Shutdown` closes it. Submission is
absent, so the default is used.

**Notes** — a rebuild wanting to know the minimum it must provide to bring the sound system up should read this
file's contract rather than any of the others.
