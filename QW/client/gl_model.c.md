# QW/client/gl_model.c

> The map and model loader for the hardware client.

**Needs** — as [`gl_model.c`](../../WinQuake/gl_model.c.md)
**Used by** — as [`gl_model.c`](../../WinQuake/gl_model.c.md)
**Tier floor** — as [`gl_model.c`](../../WinQuake/gl_model.c.md)

## Purpose

Read [`gl_model.c`](../../WinQuake/gl_model.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`gl_model.c`](../../WinQuake/gl_model.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The same two changes as [`model.c`](model.c.md) — content checksums, and tolerating a model that arrived by download — on top of the hardware loader's own work: lightmap packing, vertex list construction, skin flood-filling and texture upload, all unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
