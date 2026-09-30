# QW/client/gl_model.h

> The model records as the hardware renderer needs them.

**Needs** — as [`gl_model.h`](../../WinQuake/gl_model.h.md)
**Used by** — as [`gl_model.h`](../../WinQuake/gl_model.h.md)
**Tier floor** — as [`gl_model.h`](../../WinQuake/gl_model.h.md)

## Purpose

Read [`gl_model.h`](../../WinQuake/gl_model.h.md) for the whole substance; the algorithms are unchanged.

## State

As [`gl_model.h`](../../WinQuake/gl_model.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The networking's per-entity fields are added, as in [`model.h`](model.h.md). The lightmap page coordinates and the pre-built vertex lists are unchanged.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
