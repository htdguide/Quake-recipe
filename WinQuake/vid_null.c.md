# WinQuake/vid_null.c

> An empty video backend: publishes a buffer descriptor pointing at nothing, so a build with no display links and runs.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`vid.h`](vid.h.md)
**Used by** — the dedicated-server build
**Tier floor** — none

## Purpose

The proof that the renderer is separable. Every operation of the video interface ([`vid.h`](vid.h.md)) is present and does
nothing; the published descriptor has a zero size. A dedicated server links this, [`snd_null.c`](snd_null.c.md),
[`in_null.c`](in_null.c.md) and [`cd_null.c`](cd_null.c.md) and never touches a display.

## State

```text
VARIABLE vid : descriptor with zero dimensions
```

## The operations

**Contract** — initialize, shut down, set a mode, set or shift the palette, present a frame, and the direct-rectangle
pair: all do nothing.

**Invariants** — a rebuild should keep this file's *existence* as a design constraint: **no code above the video interface
may assume a display exists.** The null backend is the test of that, and the server half of the engine passes it. That is
what [`SYSTEM-REQUIREMENTS.md`](../SYSTEM-REQUIREMENTS.md) means by the server being viable at a higher tier alone.
