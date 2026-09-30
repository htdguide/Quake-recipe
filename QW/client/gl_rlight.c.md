# QW/client/gl_rlight.c

> Light styles, dynamic light marking, and the blob alternative.

**Needs** — as [`gl_rlight.c`](../../WinQuake/gl_rlight.c.md)
**Used by** — as [`gl_rlight.c`](../../WinQuake/gl_rlight.c.md)
**Tier floor** — as [`gl_rlight.c`](../../WinQuake/gl_rlight.c.md)

## Purpose

Read [`gl_rlight.c`](../../WinQuake/gl_rlight.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`gl_rlight.c`](../../WinQuake/gl_rlight.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Team-coloured lights are added, driven by the same two effect bits the trails use ([`r_part.c`](r_part.c.md)). Otherwise unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
