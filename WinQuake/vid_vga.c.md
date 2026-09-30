# WinQuake/vid_vga.c

> The standard-mode sub-driver: a 320 by 200 indexed framebuffer at a fixed address, presented by copy or by page flip, with the palette programmed during the vertical blank.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`vid_dos.h`](vid_dos.h.md) · [`dosisms.h`](dosisms.h.md) · [`vgamodes.h`](vgamodes.h.md) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface)
**Used by** — [`vid_dos.c`](vid_dos.c.md) fills its standard mode entry from here
**Tier floor** — T0: the framebuffer is a fixed physical address and the display controller is programmed through port registers

## Purpose

The lowest-level display path in the engine, and the only T0 code outside the assembly. It is also the clearest statement
of the two timing facts an indexed framebuffer forces on a renderer.

## State

```text
CONSTANT the display's framebuffer physical address and the standard mode number
VARIABLE vid_surfcache and the allocated buffer
VARIABLE which page is visible, in a flipped configuration
```

## `VGA_Init`, `VGA_InitMode`, `VGA_ClearVideoMem`, `VGA_FreeAndAllocVidbuffer`, `VGA_CheckAdequateMem`

**Contract** — register this driver's operations into the mode table, enter the standard mode through the firmware, clear
the display memory, and allocate the drawing buffer and the surface cache.

## `VGA_WaitVsync`

**Contract** — spins until the display controller reports that it has entered the vertical blanking interval.

**Invariants** — **two things must happen during the blank and nothing else may**: programming the palette, and switching
which page is visible. Doing either while the display is scanning shows a torn or miscoloured frame. That is the reason
this function exists and the reason it is a busy wait — there is nothing to wait *on*.

A rebuild on any platform that presents through a compositor gets both for free and deletes this. A rebuild that still
has a scanout — a retro target, an embedded display — needs it and needs to know that the wait is the *only* correct
synchronization available.

## `VGA_SetPalette`

**Contract** — waits for the blank, then writes 256 triples to the controller's palette port.

**Invariants** — the components are **six bits each**, not eight, so the engine's palette is shifted down on the way in.
That quantization is visible in gradients and is part of how the original looks. A rebuild reproducing the look must
reproduce the truncation; one that does not can use the full range.

## `VGA_SwapBuffers`, `VGA_SwapBuffersCopy`

**Contract** — present the frame either by telling the controller to display the other page, or by copying the drawing
buffer into display memory. The copy honours the engine's update rectangle.

**Invariants** — the flip is **instant and free** but gives the renderer a different buffer each frame, which breaks the
engine's dirty-rectangle tracking ([`vid_win.c`](vid_win.c.md) records the same conflict). The copy preserves the tracking
but costs the copy. Both are provided and the mode chooses. A rebuild must make the same choice consciously: **dirty
rectangles and page flipping are mutually exclusive unless you track two frames of damage.**

## `VGA_BeginDirectRect`, `VGA_EndDirectRect`

**Contract** — write a block of pixels into display memory directly and restore it.

**Notes** — this file's value to a rebuilder is the vertical-blank rule and the flip-versus-copy trade. Everything else is
a specific display controller and is a **given** that no modern platform exposes.
