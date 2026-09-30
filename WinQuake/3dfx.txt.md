# WinQuake/3dfx.txt

> Data: a note on running the hardware renderer against one vendor's accelerator through its own driver.

**Needs** — nothing
**Used by** — the reader
**Tier floor** — none

## Purpose

Installation and troubleshooting for one accelerator family: which driver library to use, the naming it must be given, and
the display-mode limitations.

## State

Data; no run-time state.

## What it records

- The renderer is used **unchanged**; only the driver library differs. So the hardware seam held against a card with a very
  different architecture from the one the renderer was written against
  ([Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer)).
- The card is **fullscreen only** and takes over the display, so the windowed modes in
  [`gl_vidnt.c`](gl_vidnt.c.md) are unavailable.
- Resolution is limited by the card's own memory, which is why the mode list is built from what the driver reports rather
  than from a fixed list.

**Notes** — historical. The transferable observation is in the first bullet: the renderer was portable across hardware that
shared only an interface, which is the argument for treating the graphics library as a seam rather than as a dependency.
