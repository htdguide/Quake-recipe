# QW/client/render.h

> The interface between the client and whichever renderer is linked.

**Needs** — as [`render.h`](../../WinQuake/render.h.md)
**Used by** — as [`render.h`](../../WinQuake/render.h.md)
**Tier floor** — as [`render.h`](../../WinQuake/render.h.md)

## Purpose

Read [`render.h`](../../WinQuake/render.h.md) for the whole substance; the algorithms are unchanged.

## State

As [`render.h`](../../WinQuake/render.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The visible-entity record gained the fields the networking needs: the remembered previous position a trail is drawn from, and the per-player skin and colour translation references ([`skin.c`](skin.c.md)). That record is the whole contract between the client and a renderer, so a rebuild swapping renderers should read it first.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
