# WinQuake/gl_vidlinux.c

> The console hardware-video backend: a rendering context on a display with no window system, with the keyboard read from the raw terminal and the mouse from a device.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`vid.h`](vid.h.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`host.c`](host.c.md) · [`gl_screen.c`](gl_screen.c.md)
**Tier floor** — T1: the terminal is put into a raw mode that must be restored, including from a signal handler

## Purpose

The same job as [`gl_vidnt.c`](gl_vidnt.c.md) with no window system at all: the display is taken over wholly. Read that
twin for the extension survey and the frame bracket. What is different here is that **every input device must be handled
directly**, and that is where the content is.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What differs

**The keyboard is the terminal, put into a raw mode.** Keys arrive as scan codes through a translation table, with press
and release distinguished by a bit. The previous terminal mode is saved and must be restored on every exit path
including signals — a process that dies holding a raw terminal leaves the user's shell unusable.

**Console switching must be intercepted.** The display is owned exclusively, so the key combination that would switch
away has to be caught, the display released, and the game suspended until the user returns. A rebuild on any platform
where the display can be taken away needs the equivalent; ignoring it means the user cannot leave.

**The mouse is a device file read for relative packets**, with the protocol variant selected by name. So relative motion
arrives natively and none of the warping in [`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md) is needed — which is the useful
comparison between the two backends.

**The display mode is chosen by searching the available modes for the requested size**, and the nearest acceptable one is
taken. There is no windowed option.

**Notes** — as with the X backend, video and input share the file because the platform gives one facility, and the same
release-on-crash rule applies to the terminal mode, the display mode and the mouse device alike.
