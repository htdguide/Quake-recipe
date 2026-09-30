# WinQuake/vid_sunxil.c

> A workstation software backend that offloads the palette-to-pixel conversion to an imaging library, optionally on a second thread.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`vid.h`](vid.h.md) · [`d_local.h`](d_local.h.md) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`host.c`](host.c.md) · [`screen.c`](screen.c.md)
**Tier floor** — T1: a second thread reads the frame buffer the renderer is writing, synchronized by hand

## Purpose

Like [`vid_x.c`](vid_x.c.md), but the conversion from indices to display pixels is expressed as an imaging-library
operation instead of a loop, and it may run on its own thread. That second point is the only thing here a rebuild should
care about, and it is worth care.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs

**The conversion and the display write can run on a second thread**, with the main loop handing off a finished frame and
continuing. Two frames' worth of buffer is needed, and a hand-built synchronization protocol decides when the renderer may
touch a buffer again.

```text
FUNCTION schedule_update(frame)
  wait until the worker is not using this buffer
  publish the frame ;  signal the worker
FUNCTION worker()
  LOOP
    wait for a published frame
    convert it and write it to the display
    mark the buffer free
FUNCTION drain()
  wait until no frame is outstanding      # before a mode change or shutdown
```

**Invariants** —

- **The renderer must not write a buffer the worker is reading.** This is the only place in the engine where two threads
  touch the frame buffer, and the whole protocol exists to prevent that one race. A rebuild that moves presentation off
  the main thread inherits the requirement exactly, and the pattern — publish, signal, wait-for-free, and an explicit
  drain before any teardown — is the right shape.
- **The drain is mandatory before a mode change or shutdown**, because the worker holds a pointer into a buffer that is
  about to be freed. Forgetting the drain gives a crash on exit that appears only sometimes.
- A **pixel-multiply check** decides whether the library's scaling path is usable, falling back when it is not.

**Notes** — the threading is optional and off by default, because the synchronization cost usually exceeded the gain at the
resolutions in play. Recorded because it is the engine's one genuine piece of multithreading and because the protocol is
the reusable part.
