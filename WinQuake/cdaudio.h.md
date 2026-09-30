# WinQuake/cdaudio.h

> The music seam: seven calls over a physical audio disc.

**Needs** — nothing beyond the base types
**Used by** — [`host.c`](host.c.md) initializes, updates and shuts it down; [`cl_parse.c`](cl_parse.c.md) and [`host_cmd.c`](host_cmd.c.md) start tracks; implemented per platform by [`cd_win.c`](cd_win.c.md), [`cd_linux.c`](cd_linux.c.md), [`cd_audio.c`](cd_audio.c.md) and [`cd_null.c`](cd_null.c.md)
**Tier floor** — none; this is the seam

## Purpose

The whole of [Seam: Redbook CD audio](../SYSTEM-REQUIREMENTS.md#seam-redbook-cd-audio), and the
shortest seam in the engine. It is also the only one that is **genuinely optional**: a complete
do-nothing implementation ships ([`cd_null.c`](cd_null.c.md)) and the engine runs against it.

## State

Stateless as declared.

## Entry points

**Contract** — `CDAudio_Init` opens the device and enumerates tracks, returning whether it succeeded.
`CDAudio_Play` starts a track by number, optionally looping. `CDAudio_Stop`, `CDAudio_Pause` and
`CDAudio_Resume` do the obvious. `CDAudio_Shutdown` releases the device. `CDAudio_Update` is called once
per frame and is where a looping track is restarted when it ends.

**Invariants** — **looping is the engine's job, not the device's.** A looping track is played once and
the per-frame update notices it has finished and plays it again. So a backend needs only "is it still
playing", not a loop mode.

The track number comes from the map's own data ([`sv_main.c`](sv_main.c.md#sv_sendserverinfo)), so a
level chooses its music by disc track.

**Notes** — the obvious modern replacement is a directory of numbered audio files, and the seam's shape
already supports it: play track *n*, stop, and report whether it is still playing. That is the whole
requirement, and a rebuild should implement exactly it rather than trying to reproduce disc access.
