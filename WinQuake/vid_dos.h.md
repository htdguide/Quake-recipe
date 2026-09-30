# WinQuake/vid_dos.h

> The interface between the DOS video dispatcher and its two sub-drivers: the mode record, and what each sub-driver must fill in.

**Needs** — [`vid.h`](vid.h.md)
**Used by** — [`vid_dos.c`](vid_dos.c.md) · [`vid_vga.c`](vid_vga.c.md) · [`vid_ext.c`](vid_ext.c.md)
**Tier floor** — none

## Purpose

Declares the mode record that makes [`vid_dos.c`](vid_dos.c.md) a dispatcher. It is the engine's only example of a
per-instance operation table, and it is worth naming as such.

## State

```text
RECORD Mode
  width, height, stride : int
  set_mode(mode)            : operation
  present(rect)             : operation
  set_palette(palette)      : operation
  begin_direct, end_direct  : operations
  the extended interface's mode number and attributes, when applicable
  description : text
VARIABLE the mode list, its count, the current index
FUNCTION the surface cache sizing rule, and the memory adequacy check
```

**Invariants** — **every mode carries its own implementation of the five operations**, so adding a display family means
adding rows, not branches. That is the structure [`vid.h`](vid.h.md) implies and this header realizes; a rebuild should keep
it, because it is the same shape as the network driver tables ([`net_bsd.c`](net_bsd.c.md)) and for the same reason.

**Notes** — the surface cache sizing rule lives here rather than in the renderer because the backend must know it to decide
whether a mode fits in memory before entering it ([`vid_win.c`](vid_win.c.md) records the same coupling).
