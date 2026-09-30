# WinQuake/glquake2.h

> An earlier copy of the hardware renderer's private header, differing only in that texture upload also takes a brightness-modulation flag.

**Needs** — [`gl_model.h`](gl_model.h.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer)
**Used by** — nothing in the shipped build
**Tier floor** — none

## Purpose

A stale duplicate of [`glquake.h`](glquake.h.md), left in the tree. Nothing includes it.

## State

The records and constants this header declares are the state; they are described in the sections below.

## What differs

The upload and load-texture operations carry an extra flag saying whether to pre-multiply the image by the overbright
factor. In the surviving header that decision moved into the library's own texture-environment setting, so the flag
disappeared from the interface.

**Notes** — recorded because the recipe mirrors the tree, and because the difference is a small piece of history worth
one sentence: pre-brightening in the upload was tried and replaced by letting the hardware do it. A rebuild uses
[`glquake.h`](glquake.h.md) and ignores this file.
