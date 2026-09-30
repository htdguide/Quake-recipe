# WinQuake/snd_linux.c

> The memory-mapped audio backend: negotiates a format with the kernel driver, maps its ring directly, and reads the consumed position from a driver query.

**Needs** — [`sound.h`](sound.h.md) · the kernel's audio device interface
**Used by** — [`snd_dma.c`](snd_dma.c.md) through [Seam: Audio output](../SYSTEM-REQUIREMENTS.md#seam-audio-output)
**Tier floor** — T2

## Purpose

The cleanest of the seven audio backends and the one that matches the seam most directly: the kernel exposes the
device's ring as mapped memory and reports how far it has consumed, which is exactly what
[`snd_dma.c`](snd_dma.c.md) wants.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## `SNDDMA_Init`

**Contract** — opens the device; honours command-line requests for a rate, a sample width and a channel count,
negotiating each with the driver and reading back what it granted; queries the ring's size; maps it; records
everything in the shared device record; and starts playback by triggering the driver. Returns whether it
succeeded, printing a diagnosis on each possible failure.

```text
FUNCTION snddma_init() -> bool
  open the audio device, reporting the reason on failure
  reset it
  # Negotiate each parameter, then read back what was actually granted.
  set the channel count (2 unless "-sndmono") ;  read it back
  set the rate (11025, or 8000/22050/44100 by the command line) ;  read back
  set the width (16 unless "-sndbits 8" or the device refuses) ;  read back
  query the ring's size
  map the ring into memory ;  RETURN false with a diagnosis on failure
  record the rate, width, channel count, sample count and address
  trigger playback
  RETURN true
```

**Invariants** — **every parameter is set and then read back**, because the driver may grant something else. A
backend that assumes what it asked for produces a mixer running at the wrong rate, which is audible as pitch error.

The ring's size comes from the driver, not from a constant, so the engine's mix-ahead is bounded by whatever the
device offers ([`snd_dma.c`](snd_dma.c.md#s_update_)).

## `SNDDMA_GetDMAPos`

**Contract** — asks the driver for the current fragment position and converts it to mono samples.

**Invariants** — this is the one call the whole sound system's timing rests on. It must advance monotonically
modulo the ring; a driver that reports a stale value produces silence or a stuck loop.

## `SNDDMA_Submit`

**Contract** — does nothing; the mapping is live.

## `SNDDMA_Shutdown`

**Contract** — unmaps the ring and closes the device.
