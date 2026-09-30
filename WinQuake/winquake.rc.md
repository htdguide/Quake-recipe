# WinQuake/winquake.rc

> Data: the Windows build's resource script — the application icon and the small dialogue shown while the engine starts.

**Needs** — [`resource.h`](resource.h.md)
**Used by** — the Windows build, through the identifiers in [`resource.h`](resource.h.md)
**Tier floor** — none

## Purpose

Declares two resources compiled into the executable.

## State

Data; no run-time state.

## What it records

```text
An ICON resource, from an image file in this directory.
A small centred, borderless DIALOG with one static text control,
  shown from program start until the game window exists.
```

**Invariants** — the dialogue exists because **the period between process start and the first drawable frame is long** —
the heap is taken, the archives are opened, the palette is read, the video mode is set — and a user with no feedback assumes
nothing happened. It is dismissed once video is up, after which [`gl_screen.c`](gl_screen.c.md) or
[`screen.c`](screen.c.md) can draw a loading indicator of its own.

A rebuild needs the same two-stage loading feedback: something the platform can show before the renderer exists, then the
renderer's own.

**Notes** — the icon and the rest of the script are build artifacts with no design content.
