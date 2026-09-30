# QW/client/gl_vidlinux_x11.c

> The windowed hardware-video backend, renamed.

**Needs** — as [`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md)
**Used by** — the windowed hardware build
**Tier floor** — as [`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md)

## Purpose

The same file as [`gl_vidlinuxglx.c`](gl_vidlinuxglx.c.md) under a clearer name. Read that twin, and
[`gl_vidlinuxglx.c`](../../WinQuake/gl_vidlinuxglx.c.md) for the substance — in particular the pointer grab and the
warp-to-centre hazard.

**Notes** — a checked-in rename, as above.
## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

