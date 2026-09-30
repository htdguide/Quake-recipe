# WinQuake/vid_x.c

> The X11 software-video backend: the engine's indexed buffer converted to the display's own pixel format on every present, through a shared-memory image when the server is local.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`vid.h`](vid.h.md) · [`d_local.h`](d_local.h.md) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`host.c`](host.c.md) · [`screen.c`](screen.c.md)
**Tier floor** — none

## Purpose

The one software backend where the display is **not** indexed, so the engine's palette indices must be translated to real
pixels every frame. That translation is the whole content of the file and the thing a rebuild on any true-colour display
must reproduce.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What is worth recording

**A conversion pass per present, one function per display depth.** For each supported depth — 8, 16 and 24 bits per pixel,
and a 16-bit variant needing byte fix-ups — there is a loop that walks the dirty rectangle, looks each index up in a table
of pre-converted pixels, and writes it in the display's format.

```text
FUNCTION present(rect)
  FOR EACH row, FOR EACH column in rect
    out[x] = pixel_table[ buffer[x] ]        # pixel_table built from the palette
  hand the image to the display
```

**Invariants** —

- **The palette becomes a lookup table of display pixels**, rebuilt whenever the palette changes. So a palette change
  still costs only 256 conversions, and the engine's brightness effects stay cheap — the property
  [`vid_dos.c`](vid_dos.c.md) gets for free is preserved here by construction. This is the single most important
  adaptation in the file and it generalizes: **an indexed renderer runs on a true-colour display at the cost of one table
  lookup per pixel, and none of its palette tricks have to change.**
- On an indexed display the table is the identity and the palette is programmed instead, so the 8-bit path is a copy.
- The 16-bit variants exist because some servers wanted the two halves of each pixel in an order the straightforward loop
  does not produce; the fix-up passes correct them in place.

**A shared-memory image is used when the display is local**, avoiding a copy through the protocol, and the code must fall
back to the ordinary path when it is not — detected, not assumed. Failing to detect gives a working game only on the
developer's machine.

**Mouse look needs the pointer grabbed, hidden and warped back to the centre**, exactly as in
[`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md) — read that twin for the warp-event hazard.

**The window's dithering can be toggled**, and the console input is read from the terminal in parallel with the window's
events, which is what lets a server be driven from the shell while a client window is open.

**Notes** — the file also carries a handful of operations named after an unrelated program's window interface, left from
the code this backend was adapted from. They are unused.
