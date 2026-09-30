# WinQuake/vid_dos.c

> The DOS software-video backend: a mode table spanning the standard low-resolution mode and whatever the display's extended interface offers, dispatching each operation to whichever sub-driver owns the current mode.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`vid.h`](vid.h.md) · [`vid_dos.h`](vid_dos.h.md) · [`dosisms.h`](dosisms.h.md) · [`d_local.h`](d_local.h.md) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface)
**Used by** — [`host.c`](host.c.md) · [`screen.c`](screen.c.md)
**Tier floor** — T1: the framebuffer is memory at an address the display controller owns, and modes are entered through firmware calls

## Purpose

The original platform, and the one whose constraints shaped the video interface ([`vid.h`](vid.h.md)). It is a dispatcher:
the mode table's entries each name their own set-mode, present, set-palette and direct-rectangle operations, filled by
[`vid_vga.c`](vid_vga.c.md) for the standard mode and [`vid_ext.c`](vid_ext.c.md) for the extended ones.

## State

```text
RECORD Mode
  width, height, stride : int
  set_mode, present, set_palette, begin_direct, end_direct : operations
  extended-interface mode number and its attributes
  description : text
VARIABLE modelist, numvidmodes, vid_modenum, vid_default
VARIABLE vid : the published buffer descriptor
```

**Invariants** — **each mode carries its own implementation.** That is the whole structure, and it is why the interface in
[`vid.h`](vid.h.md) is shaped as a set of operations over a descriptor rather than as a class: this table *is* the
polymorphism.

The published buffer may be the display's own memory, or a buffer in ordinary memory that is copied to the display on
present — the mode decides. So the renderer cannot know whether it is drawing to the visible screen, which is another
reason the buffer's address is re-read rather than cached.

## `VID_Init`, `VID_Shutdown`, `VID_SetMode`, `VID_SetDefaultMode`

**Contract** — survey the display: always add the standard low-resolution mode, then ask the extended interface for every
mode it supports at one byte per pixel and add the usable ones; pick a default from the command line or the largest that
fits memory; then set it. Shutdown restores the text mode.

**Invariants** — **the text mode must be restored on every exit path**, including the fatal one
([`sys_dos.c`](sys_dos.c.md)), or the user is left looking at a graphics mode with a shell running behind it.

## `VID_Update`, `VID_SetPalette`, `VID_ShiftPalette`

**Contract** — present through the current mode's operation; program the palette; and apply a brightness shift by
reprogramming the palette.

**Invariants** — **brightness is a palette operation, not a per-pixel one.** The engine's damage flash and its gamma both
work by reprogramming 256 entries, which is free compared with touching every pixel. That is the single biggest advantage
of an indexed framebuffer and the reason the engine is built on one; a rebuild in true colour pays for both effects per
pixel and should know it is paying.

## `VID_GetModePtr`, `VID_NumModes`, `VID_ModeInfo`, `VID_GetModeDescription`, and the four describe commands plus the test command

**Contract** — expose the mode table to the console.

## `D_BeginDirectRect`, `D_EndDirectRect`, `VID_MenuDraw`, `VID_MenuKey`

**Contract** — dispatch the direct-rectangle pair to the current mode, and the video menu page.

**Notes** — the interesting content of the DOS video family is not here but in the two sub-drivers: the page-flip and
vertical-blank handling in [`vid_vga.c`](vid_vga.c.md), and the bank-switched addressing in
[`vid_ext.c`](vid_ext.c.md). This file is the table that selects between them.
