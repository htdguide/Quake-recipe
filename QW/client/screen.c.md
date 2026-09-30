# QW/client/screen.c

> Composing the frame: the view rectangle, the console, the centre prints, the frame-rate display, and the screen capture the server can request.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`draw.h`](draw.h.md) · [`vid.h`](vid.h.md) · [`sbar.h`](sbar.h.md)
**Used by** — [`cl_main.c`](cl_main.c.md) once per frame
**Tier floor** — none

## Purpose

Read [`screen.c`](../../WinQuake/screen.c.md) for the whole substance: the view sizing, the field-of-view correction, the dirty-rectangle
tracking, the centre-print queue and the console height.

## State

As [`screen.c`](../../WinQuake/screen.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**The loading plaque is gone.** The original stops the world and displays a plaque while a level loads
([`screen.c`](../../WinQuake/screen.c.md)); here a level change keeps the connection alive and the client keeps rendering, so there is
nothing to block on. That absence is a direct consequence of the persistent-server design
([`cl_main.c`](cl_main.c.md)).

**A frame-rate display is added**, averaged over a window. Worth noting because the frame rate is now also the packet rate
([`cl_input.c`](cl_input.c.md)), so it is a network diagnostic as much as a rendering one.

**A screen capture can be requested by the server** and is reduced and encoded here before being uploaded
([`cl_parse.c`](cl_parse.c.md)). It is reduced heavily — the image is shrunk and colour-matched to a coarse palette — because it must
fit through a game-speed link. The player can refuse ([`sv_user.c`](../server/sv_user.c.md)); the recipe records the capability as
privacy-relevant and the refusal as the mitigation.

**Text can be drawn into the capture buffer**, so a capture is annotated with who took it and when — which is the point of the
facility, since it exists to settle accusations.

**Invariants** — the colour matching from the framebuffer's palette to the capture's smaller one is a **nearest-colour search per
distinct value**, cached, because a per-pixel search would be far too slow. A rebuild reducing an image should do the same.

**Notes** — the dirty-rectangle tracking is still here in the software build and still absent from the hardware one
([`gl_screen.c`](gl_screen.c.md)) — the same split as the original.
