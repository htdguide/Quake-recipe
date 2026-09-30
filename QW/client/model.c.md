# QW/client/model.c

> The map and model loader for the software client.

**Needs** — as [`model.c`](../../WinQuake/model.c.md)
**Used by** — as [`model.c`](../../WinQuake/model.c.md)
**Tier floor** — as [`model.c`](../../WinQuake/model.c.md)

## Purpose

Read [`model.c`](../../WinQuake/model.c.md) for the whole substance; the algorithms are unchanged.

## State

As [`model.c`](../../WinQuake/model.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

Two changes. Entities record their leaf lists as before, but the client now also needs a model's checksum to report to the server ([`sv_init.c`](../server/sv_init.c.md)), so the loader computes one. And the loader must tolerate being called for a model that arrived by download after the level began ([`cl_parse.c`](cl_parse.c.md)). The three precompiled collision hulls and their hard-coded body sizes are unchanged, and the client now uses them itself for prediction ([`pmovetst.c`](pmovetst.c.md)).

**Notes** — recorded because the recipe mirrors the tree. Where a delta is purely mechanical it is noted as such, so a reader can
skip to the original twin.
