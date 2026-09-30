# WinQuake/vid_svgalib.c

> The console software-video backend: a display taken over through a graphics library, a byte-per-pixel buffer copied out each frame, with the keyboard read raw and the mouse from a device.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`vid.h`](vid.h.md) · [`d_local.h`](d_local.h.md) · [Seam: Framebuffer surface](../SYSTEM-REQUIREMENTS.md#seam-framebuffer-surface) · [Seam: Keyboard and mouse input](../SYSTEM-REQUIREMENTS.md#seam-keyboard-and-mouse-input)
**Used by** — [`host.c`](host.c.md) · [`screen.c`](screen.c.md)
**Tier floor** — T1: the terminal and the display are both taken over and must be restored from a signal handler

## Purpose

The software renderer on a machine with no window system. Structurally it is [`vid_dos.c`](vid_dos.c.md)'s job done through
a library instead of firmware, plus the input handling that [`gl_vidlinux.c`](gl_vidlinux.c.md) also does. Read those two
twins; this one records what is specific.

## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

## What is worth recording

**The mode list comes from the library, filtered to indexed modes, and a pixel-doubling option is offered** — drawing at
half the resolution and presenting each pixel as a block. That is the same trade the engine offers everywhere: fill rate
for sharpness, exposed as a setting.

**Presentation is a copy of the dirty rectangle**, or the library's own block copy for a doubled mode. The dirty tracking
therefore works, as in the banked DOS path.

**Gamma is a console command that reprograms the palette**, not a startup-only setting — the first backend where it is
live, because reprogramming a palette is cheap and no textures depend on it (unlike the hardware path,
[`gl_vidnt.c`](gl_vidnt.c.md#check_gamma)).

**Console switching is intercepted and the game suspended**, and the display, the terminal mode and the mouse device are
all released on that path and on every fatal signal. Same rule as
[`gl_vidlinux.c`](gl_vidlinux.c.md): what you take from the system you release on the crash path.

**The keyboard is raw scan codes through a translation table with press and release separated**, and the mouse is a device
read for relative packets with the protocol selected by name — so no pointer warping is needed.

**Notes** — video and input share the file for the same platform reason as the other console backends. The engine's own
interfaces ([`vid.h`](vid.h.md), [`input.h`](input.h.md)) stay separate, which is what matters.
