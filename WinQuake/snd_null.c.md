# WinQuake/snd_null.c

> A complete do-nothing sound system, compiled in place of the real one for a build with no audio.

**Needs** — [`sound.h`](sound.h.md)
**Used by** — the dedicated-server and no-sound builds, replacing [`snd_dma.c`](snd_dma.c.md), [`snd_mix.c`](snd_mix.c.md) and [`snd_mem.c`](snd_mem.c.md)
**Tier floor** — none

## Purpose

Not a backend — a stub for the **whole sound system**. It implements every entry point
[`sound.h`](sound.h.md) declares as nothing, so that a build with no audio links and runs.

Its existence is the evidence that sound is genuinely optional in this engine: the dedicated server uses it, and
nothing else in the tree changes.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## Contract

**Contract** — every declared operation does nothing. The precache functions return nothing, and the record
pointers stay unset.

**Invariants** — the callers must therefore tolerate a null sound record everywhere, which they do
([`cl_parse.c`](cl_parse.c.md) passes whatever the precache returned straight to the start-sound call, and that
call is also a stub here).

**Notes** — the same pattern appears for video ([`vid_null.c`](vid_null.c.md)), input
([`in_null.c`](in_null.c.md)), music ([`cd_null.c`](cd_null.c.md)) and networking
([`net_none.c`](net_none.c.md)). Five subsystems with complete stubs is what makes a headless build of this engine
a link-time decision rather than a set of conditionals — and it is worth reproducing.
