# WinQuake/d_sprite.c

> Rasterizes a sprite: scans its clipped polygon into spans, then fills them with perspective-corrected sampling, a transparency test and a depth test.

**Needs** — [`d_local.h`](d_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`r_local.h`](r_local.h.md)
**Used by** — [`r_sprite.c`](r_sprite.c.md) fills the sprite descriptor and calls; [`d_spr8.s`](d_spr8.s.md) is the accelerated filler
**Tier floor** — none

## Purpose

A sprite is a convex polygon — usually a quad — with a texture, drawn after the world so that it can
depth-test against it. This file scans it and fills it. The scan is a self-contained miniature of the edge
sorter's job for a single convex shape, which is why it does not go through the edge list at all.

## State

```text
VARIABLE sprite_height : int
VARIABLE minindex, maxindex : int       # the topmost and bottommost vertices
VARIABLE sprite_spans : SSpan           # the scanned span list
```

## `D_SpriteScanLeftEdge`

**Contract** — walks the polygon's vertices from the topmost toward the bottommost in one direction,
emitting one span start per scanline crossed. Uses a fixed-point x stepped by the edge's slope.

```text
FUNCTION d_sprite_scan_left_edge()
  i = minindex, wrapping to the vertex count when zero
  vtop = ceil(vertex[i].v)
  REPEAT
    pvert = vertex[i] ;  pnext = the PREVIOUS vertex      # walking backwards
    vbottom = ceil(pnext.v)
    IF vtop < vbottom
      slope = (pnext.u - pvert.u) / (pnext.v - pvert.v)
      u_step = truncate(slope * 65536)
      # Start x at the edge's position at the first whole scanline, then bias
      # up so that the shift-right rounds toward the ceiling.
      u = truncate((pvert.u + slope*(vtop - pvert.v)) * 65536) + 0xFFFF
      FOR EACH scanline v FROM vtop TO vbottom-1
        emit a span start AT (u SHIFTED RIGHT 16, v)
        u = u + u_step
    vtop = vbottom
    step i backwards, wrapping
  UNTIL i reaches maxindex
```

**Invariants** — **the scanline is taken at the ceiling of the vertex's y** and the x is biased by one less
than a whole unit so that the right shift rounds up. Both are the standard top-left fill convention, and
getting either wrong makes adjacent sprites overlap or leave a seam.

The walk is backwards through the vertex list because the polygon's winding puts the left edge in that
direction. A right-edge counterpart computes the counts.

## `D_SpriteDrawSpans`

**Contract** — fills a list of sprite spans from the cached sprite texture, with eight-pixel perspective
subdivision, skipping transparent texels and depth-testing each written pixel. The list is terminated by a
span whose count is the sentinel value.

```text
FUNCTION d_sprite_draw_spans(spans)
  pbase = the sprite's pixels
  precompute the eight-pixel gradient advances, and the per-pixel depth step
  FOR EACH span UNTIL the terminator
    IF its count is not positive  CONTINUE
    compute the exact s, t and depth at the span's start, clamped
    LOOP over eight-pixel sub-spans, exactly as d_draw_spans_8 does
      REPEAT spancount TIMES
        texel = pbase[(s SHIFTED RIGHT 16) + (t SHIFTED RIGHT 16)*cachewidth]
        IF texel != 255                                   # NOT transparent
          IF the stored depth <= (izi SHIFTED RIGHT 16)
            store the depth ;  store the texel
        izi = izi + izistep                               # depth steps PER PIXEL
        s = s + sstep ;  t = t + tstep
```

**Invariants** — three differences from the world filler, and all three matter.

**The transparency test is per pixel**, against palette index 255. That costs a compare and a branch for
every texel and is why sprites are the most expensive pixels in the renderer. It also means a sprite's
bounding quad can be mostly empty without cost beyond the test.

**Depth is stepped per pixel and tested per pixel**, where the world filler writes depth in a separate pass.
A sprite must test rather than merely write, because it is drawn after the world.

**A sprite writes the depth buffer where it draws**, so two overlapping sprites occlude correctly — but
only where they are opaque, which is what makes overlapping explosions composite the way they do.

The span list is terminated by a **sentinel count** rather than by a null link
([`d_local.h`](d_local.h.md)), because the spans are a flat array here rather than a linked list.

The subdivision interval, the clamping and the low bound of 8 are identical to
[`d_scan.c`](d_scan.c.md#d_drawspans8) and the reasons are the same.
