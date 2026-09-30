# QW/client/model.h

> The model records.

**Needs** — as [`model.h`](../../WinQuake/model.h.md)
**Used by** — as [`model.h`](../../WinQuake/model.h.md)
**Tier floor** — as [`model.h`](../../WinQuake/model.h.md)

## Purpose

Read [`model.h`](../../WinQuake/model.h.md) for the whole substance; the algorithms are unchanged.

## State

As [`model.h`](../../WinQuake/model.h.md); the records are unchanged except where **What differs** says otherwise.

## What differs

The per-entity leaf list and the checksum field are added. The three collision hulls are unchanged and are now read by the client's own collision queries.

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
