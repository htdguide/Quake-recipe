# WinQuake/cd_linux.c

> The Unix music backend: the same track-playing interface through a device's control operations.

**Needs** — [`quakedef.h`](quakedef.h.md) · [Seam: Redbook CD audio](../SYSTEM-REQUIREMENTS.md#seam-redbook-cd-audio)
**Used by** — [`cl_parse.c`](cl_parse.c.md) · [`host.c`](host.c.md)
**Tier floor** — none

## Purpose

Read [`cd_win.c`](cd_win.c.md) for the interface and the four behavioural rules — silent failure, track remapping, no-op
re-play, and looping by detection. They are identical.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs

Track positions are expressed in the disc's own minute-second-frame form rather than as plain track numbers, so a play
request must read the track table and convert. Completion is discovered by **polling the drive's status** rather than by a
notification, so the frame poll is doing real work here.

The device name is a setting, and opening it may fail for permission reasons — which is a common, recoverable, and silent
failure.

**Notes** — the conversion between a track number and a disc position is the only added content, and it is a property of the
medium a rebuild will not have.
