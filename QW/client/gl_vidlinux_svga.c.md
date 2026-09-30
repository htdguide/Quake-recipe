# QW/client/gl_vidlinux_svga.c

> The console hardware-video backend, renamed.

**Needs** — as [`gl_vidlinux.c`](gl_vidlinux.c.md)
**Used by** — the console hardware build
**Tier floor** — as [`gl_vidlinux.c`](gl_vidlinux.c.md)

## Purpose

The same file as [`gl_vidlinux.c`](gl_vidlinux.c.md) under a name that says which display path it takes. Read that twin, and
[`gl_vidlinux.c`](../../WinQuake/gl_vidlinux.c.md) for the substance.

**Notes** — the two copies in one directory are a checked-in rename, not a variant. Recorded because the recipe mirrors the tree.
## State

The backend's own handles, buffers and device state, as the sections below describe. Nothing above the interface sees any of it.

