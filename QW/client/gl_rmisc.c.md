# QW/client/gl_rmisc.c

> Hardware renderer startup, the particle texture, and per-player skin translation.

**Needs** — as [`gl_rmisc.c`](../../WinQuake/gl_rmisc.c.md)
**Used by** — as [`gl_rmisc.c`](../../WinQuake/gl_rmisc.c.md)
**Tier floor** — as [`gl_rmisc.c`](../../WinQuake/gl_rmisc.c.md)

## Purpose

Read [`gl_rmisc.c`](../../WinQuake/gl_rmisc.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`gl_rmisc.c`](../../WinQuake/gl_rmisc.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The skin translation now takes its source from the shared downloaded skin where a player declared one ([`skin.c`](skin.c.md)) and from the model's own pixels otherwise. The reserved handle range per player, the palette range remapping and the reversed upper ramps are unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
