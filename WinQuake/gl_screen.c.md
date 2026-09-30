# WinQuake/gl_screen.c

> The frame composer for the hardware renderer: the same view sizing, console, loading plaque and notifications, bracketed by the library's frame calls and with the screen capture read back from the framebuffer.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`gl_draw.c`](gl_draw.c.md) · [`gl_rmain.c`](gl_rmain.c.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer)
**Used by** — [`host.c`](host.c.md) calls the update each frame
**Tier floor** — none

## Purpose

Read [`screen.c`](screen.c.md) for the whole of the logic: how the view rectangle is derived from the size setting and
the status bar, how the field of view is corrected for a non-standard aspect, the centre-print queue, the loading plaque
and its re-entrancy guard, the modal question, and the rule that the renderer is never called from inside a load. All of
it is unchanged. This twin records the differences.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## What differs

**The frame is bracketed by two platform calls.** The update begins by asking the platform to make the context current
and report the viewport, and ends by asking it to present ([`glquake.h`](glquake.h.md)). Those two calls are the entire
platform surface of the hardware renderer, which is why the seam is so small
([Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer)).

**There is no dirty-rectangle bookkeeping.** The software path tracks which parts of the screen changed so it can update
only those ([`screen.c`](screen.c.md), [`draw.c`](draw.c.md)); here the whole frame is redrawn and presented every time,
because presenting is all-or-nothing. Every `scr_copy` and update-region variable in the software version has no
counterpart, and a rebuild on any presenting API drops the concept.

**The border around a reduced-size view is filled explicitly** by four tiling fills, once per frame, rather than being
left alone because it did not change. That is the direct consequence of having no dirty tracking.

**The screen capture reads the framebuffer back** and writes an uncompressed true-colour image with a hand-built
eighteen-byte header, rather than converting the indexed framebuffer to a palette image. It captures the window's
viewport, not the engine's logical resolution, so a capture is at the real resolution.

**Invariants** — the readback is bottom-up, which matches the image format's default row order, so no flip is needed.
A rebuild whose readback or whose format differs in row order must flip, and the symptom is obvious.

**The 2D mode is entered once per frame** before any interface drawing ([`gl_draw.c`](gl_draw.c.md#gl_set2d)), where the
software path needed no mode at all.

**Notes** — the shared portion of these two files is large and the differing portion is small and mechanical, which is
itself the useful observation: the frame composer is almost renderer-independent, and would be entirely so given a
`begin frame` / `end frame` / `fill rectangle` interface. A rebuild should factor it that way and have one copy.
