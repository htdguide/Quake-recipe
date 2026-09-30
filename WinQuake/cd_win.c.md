# WinQuake/cd_win.c

> The Windows music backend: plays numbered audio tracks from the game disc, remembers a looping track, and pauses with the game.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`winquake.h`](winquake.h.md) · [Seam: Redbook CD audio](../SYSTEM-REQUIREMENTS.md#seam-redbook-cd-audio)
**Used by** — [`cl_parse.c`](cl_parse.c.md) starts a track on map load; [`host.c`](host.c.md) polls it
**Tier floor** — none

## Purpose

The engine's music. The original ships its soundtrack as audio tracks on the game disc, so "play music" means "play track
*n* of whatever disc is in the drive". A rebuild will substitute files
([Seam: Redbook CD audio](../SYSTEM-REQUIREMENTS.md#seam-redbook-cd-audio)), and the only things it needs from this file are the interface
and four behavioural rules.

## State

```text
VARIABLE enabled, initialized, playing, wasPlaying, playLooping
VARIABLE playTrack, maxTrack, remap[100]
VARIABLE the device handle, and the current volume
```

## `CDAudio_Init`, `CDAudio_Shutdown`, `CDAudio_GetAudioDiskInfo`

**Contract** — open the drive in track-number mode, read how many tracks the disc has, register the command; and stop and
close. Report failure without preventing the game from running.

**Invariants** — **a missing drive, a missing disc or a data-only disc disables music and nothing else.** Music is the one
subsystem whose absence must be completely silent, because most players did not keep the disc in the drive. Every failure
path here therefore clears an enabled flag rather than reporting an error.

## `CDAudio_Play`, `CDAudio_Stop`, `CDAudio_Pause`, `CDAudio_Resume`

**Contract** — play a track, optionally looping; stop; pause remembering that it was playing; resume if it was.

**Invariants** —

- **A track number is remapped through a table** the player can set, so a soundtrack shipped on a different disc layout
  can be used. A remapped entry of zero means "no music for this track".
- A track that does not exist, or is a data track, is **ignored silently**.
- Playing the track that is already playing is a **no-op**, because map loads re-issue the same request and restarting
  would stutter.
- The pause must remember whether it was playing, so resuming does not start music that was never on.

## `CDAudio_Update`, `CDAudio_MessageHandler`

**Contract** — polled each frame: apply the current volume setting, and restart the track when the drive reports it has
finished and looping was requested. The platform's completion notification arrives as a window message.

**Invariants** — **looping is done by detecting the end and starting again**, not by asking the drive to loop, because the
drive cannot. So the poll is mandatory and the gap between tracks is audible. A rebuild playing a file should loop
seamlessly and will sound better than the original.

Volume is applied every frame rather than on change, which is wasteful and harmless; a rebuild applies it on change.

## `CD_f`, `CDAudio_Eject`, `CDAudio_CloseDoor`

**Contract** — the console command: on, off, reset, remap, play, loop, stop, pause, resume, eject, close, info.

**Notes** — the eject and door commands exist because the player must sometimes take the disc out, and the engine holds the
drive open. A rebuild has no equivalent and should drop them.
