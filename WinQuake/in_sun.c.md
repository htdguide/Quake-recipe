# WinQuake/in_sun.c

> The workstation input backend: mouse motion read from the window system's pointer, re-centred each frame; no joystick.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`input.h`](input.h.md) · [`client.h`](client.h.md) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`cl_input.c`](cl_input.c.md) · [`host.c`](host.c.md)
**Tier floor** — none

## Purpose

The minimum filling of the input seam. Read [`in_win.c`](in_win.c.md) for the accumulate-and-consume rule and the pitch
clamp. Here the mouse's position is read, the offset from the window centre taken as the motion, and the pointer warped
back — the same technique and the same hazard as [`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md).

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs

No joystick, no direct device, no acceleration handling, and no filter. Keys arrive through the video backend's event loop
([`vid_sunx.c`](vid_sunx.c.md)), which is where this platform's keyboard lives.

**Notes** — worth one page as the floor: the engine plays with nothing but relative mouse motion and a key state table.
Everything else in the input chapter is refinement.
