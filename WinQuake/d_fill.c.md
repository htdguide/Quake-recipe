# WinQuake/d_fill.c

> Fills a rectangle of the display with one palette index, clipped to the surface.

**Needs** — [`vid.h`](vid.h.md) · [`quakedef.h`](quakedef.h.md)
**Used by** — [`draw.c`](draw.c.md) · the video backends
**Tier floor** — none

## Purpose

A rectangle fill in palette indices, used by the two-dimensional layer and by the video backends to clear regions.

## State

Stateless; it writes through the rasterizer's shared destination pointers ([`d_local.h`](d_local.h.md)).

## `D_FillRect`

**Contract** — takes a rectangle and a palette index; clips the rectangle to the surface and fills it,
writing four bytes at a time where both the width and the destination are four-byte aligned. An empty
rectangle after clipping does nothing.

```text
FUNCTION d_fill_rect(rect, color)
  clip rect's left and top to zero, adjusting its extent
  clip its right to the surface's width
  clip its bottom to the surface's height        # see the note
  IF the width or height is now under 1  RETURN
  IF the width and the destination are both 4-byte aligned
    replicate `color` into a 32-bit word and store one per four pixels
  ELSE
    store one byte per pixel
```

**Invariants** — the bottom clip **subtracts the x coordinate rather than the y**, which is a real bug: a
rectangle extending past the bottom of a surface is clipped by the wrong amount and can write past the
end. Reachable only from a caller passing an oversized rectangle, and none does. A rebuild should write
the correct test.

**Notes** — the four-at-a-time path requires the *width* to be a multiple of four as well as the address,
which is stricter than needed — a partial tail could be handled separately. Incidental.
