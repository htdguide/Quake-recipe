# WinQuake/snd_sun.c

> An audio backend for a platform that reports an absolute sample count rather than a ring position.

**Needs** — [`sound.h`](sound.h.md) · the platform's audio device
**Used by** — [`snd_dma.c`](snd_dma.c.md) through [Seam: Audio output](../SYSTEM-REQUIREMENTS.md#seam-audio-output)
**Tier floor** — T2

## Purpose

Notable for one reason: this platform's driver reports **how many samples it has played since the device opened**,
not where in the ring it is. So the backend offers an extra entry point that returns that absolute count directly,
and the engine's clock ([`snd_dma.c`](snd_dma.c.md#getsoundtime)) uses it in place of its own wrap counting.

That is the better interface, and a rebuild should ask its device for the same thing.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## `SNDDMA_Init`

**Contract** — opens the device, sets the format, allocates a host buffer, and fills in the device record.
Returns whether it succeeded.

## `SNDDMA_GetSamples`

**Contract** — returns the absolute count of samples the device has played.

**Invariants** — **this is the extra entry point**, selected by a build switch in the engine's clock. With it, the
wrap counting, the ring-length modulo and the near-overflow reset in
[`snd_dma.c`](snd_dma.c.md#getsoundtime) are all unnecessary — and so is the comment admitting that a double wrap
between calls is miscounted.

## `SNDDMA_GetDMAPos`

**Contract** — derives a ring position from the absolute count, for the code paths that still want one.

## `SNDDMA_Submit`, `SNDDMA_Shutdown`

**Contract** — write the newly-mixed region to the device; and close it.
