# QW/client/zone.h

> Identical to the original engine's copy; read that twin.

**Needs** — as [`zone.h`](../../WinQuake/zone.h.md)
**Used by** — as [`zone.h`](../../WinQuake/zone.h.md)
**Tier floor** — as [`zone.h`](../../WinQuake/zone.h.md)

## Purpose

Byte-for-byte the same file as [`WinQuake/zone.h`](../../WinQuake/zone.h.md). The allocators' interface, unchanged — the double-ended stack, the small heap and the evictable cache are the same.

## State

As [`zone.h`](../../WinQuake/zone.h.md).

**Notes** — recorded because the recipe mirrors the tree. The duplication is in the source, not in the design: one file, compiled
into two programs from two directories.
