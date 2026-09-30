# WinQuake/cd_audio.c

> The DOS music backend: the same track-playing interface driven through the disc driver's request packets, with the disc's own position arithmetic done by hand.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`dosisms.h`](dosisms.h.md) · [Seam: Redbook CD audio](../SYSTEM-REQUIREMENTS.md#seam-redbook-cd-audio)
**Used by** — [`cl_parse.c`](cl_parse.c.md) · [`host.c`](host.c.md)
**Tier floor** — T1: requests are structures written into memory the driver reads directly

## Purpose

Read [`cd_win.c`](cd_win.c.md) for the interface and the rules. This is the same thing over a driver reached by writing
request packets, in the same style as [`net_bw.c`](net_bw.c.md).

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What is worth recording

**Every operation is a request packet.** A command code, the parameters, and a status field the driver fills in. Reading the
track table, querying the status, starting a play and stopping are each one packet.

**Disc positions are a three-part value and the arithmetic is explicit.** Converting a position to an absolute frame count
means combining minutes, seconds and frames with the medium's fixed rates and subtracting the lead-in offset. That
conversion is the only real computation in the file:

```text
FUNCTION position_to_frame(minutes, seconds, frames) -> int
  RETURN ((minutes * 60) + seconds) * frames_per_second + frames - lead_in_frames
```

**Media change must be detected**, because the player can swap the disc while the game runs, and every cached track number
becomes wrong. The check is a separate request and is made before a play.

**Invariants** — the lead-in offset is **not optional**: omitting it plays each track a fixed distance into the wrong place.
The frame rate is fixed by the medium.

**Notes** — nothing here survives a rebuild except the observation that media can change under you, which generalizes to any
removable or user-replaceable content source.
