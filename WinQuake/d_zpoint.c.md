# WinQuake/d_zpoint.c

> Writes one depth-tested pixel.

**Needs** — [`d_local.h`](d_local.h.md) · [`d_iface.h`](d_iface.h.md)
**Used by** — [`d_polyse.c`](d_polyse.c.md), for the point-plotting path that replaces triangle rasterization on distant models
**Tier floor** — none

## Purpose

One pixel, depth-tested, for the particle path. It exists as its own file because the hand-written version replaces it
([Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)) and the two must be swappable.

## State

Stateless; it reads the rasterizer's shared destination pointers ([`d_local.h`](d_local.h.md)).

## `D_DrawZPoint`

**Contract** — reads the point descriptor global; writes its colour and depth at its screen position if the
depth buffer admits it.

```text
FUNCTION d_draw_z_point()
  pz    = the depth buffer AT (r_zpointdesc.v, r_zpointdesc.u)
  pdest = the view buffer  AT the same place
  izi = truncate(r_zpointdesc.zi * 0x8000)
  IF the stored depth <= izi                     # nearer or equal wins
    store izi ;  store the colour
```

**Invariants** — the test is **less than or equal**, so a point at exactly the stored depth overwrites. That
matters for the point-plotting model path, where several of a model's vertices can land on the same pixel
at the same depth and the later one should win — which keeps the model's surface coherent rather than
speckled.

The depth scaling by `0x8000` differs from the span filler's `0x8000 * 0x10000`
([`d_scan.c`](d_scan.c.md#d_drawzspans)) because the descriptor's reciprocal is already in a different
scale. A rebuild should use one representation throughout.

No bounds check at all: the caller must have clipped.
