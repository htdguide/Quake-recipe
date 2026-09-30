# WinQuake/README.Solaris

> Data: the workstation port's release note — what it needs, what it omits, and the known problems.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

A short document whose value to the recipe is its list of omissions, because each one names a capability the engine can do
without.

## State

Data; no run-time state.

## What it records

- The port runs the **software renderer only**, in a window, with no hardware path.
- It needs a shared-memory extension for acceptable speed, and falls back without it
  ([`vid_sunx.c`](vid_sunx.c.md)).
- **No music** ([`cd_null.c`](cd_null.c.md)), and the sound device support is limited to specific models
  ([`snd_sun.c`](snd_sun.c.md)).
- Performance expectations, and the note that the pixel-doubling option is how a slower machine copes — the same trade
  every video backend offers.

**Notes** — treat the omission list as a **minimum viable engine**: software renderer, a window, a keyboard, a mouse, and no
music. A rebuild can ship exactly that and be playable, which is a useful thing to know before starting.
