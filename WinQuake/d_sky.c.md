# WinQuake/d_sky.c

> Draws the sky: for each span, reverse-projects screen pixels onto a fixed-radius sphere and samples the scrolling two-layer sky texture.

**Needs** — [`d_local.h`](d_local.h.md) · [`r_local.h`](r_local.h.md) · [`vid.h`](vid.h.md)
**Used by** — [`d_edge.c`](d_edge.c.md), for surfaces flagged as sky
**Tier floor** — none

## Purpose

The sky is not geometry. It is a texture sampled by **reverse-projecting each screen pixel into a
direction** and using that direction's horizontal components as texture coordinates — so the sky appears
infinitely distant and parallax-free, and it scrolls with time.

## State

```text
CONSTANT sky_span_shift = 5 ;  sky_span_max = 32
```

**Invariants** — thirty-two pixels between exact computations, four times the interval the world uses
([`d_scan.c`](d_scan.c.md)). The sky's mapping is far less perspective-sensitive — it is a direction, not a
plane — so a coarser subdivision is invisible.

## `D_Sky_uv_To_st`

**Contract** — takes a screen pixel; returns the sky texture coordinate at that pixel, in 16.16 fixed
point, including the time-based scroll.

```text
FUNCTION d_sky_uv_to_st(u, v) -> (s, t)
  # Normalize the screen position by the LARGER view dimension, so the sky's
  # scale does not change with aspect ratio.
  temp = max(view width, view height)
  wu = 8192 * (u - display width/2)  / temp
  wv = 8192 * (display height/2 - v) / temp          # y inverted

  # Reverse-project: a direction at a nominal depth of 4096.
  end = 4096*vpn + wu*vright + wv*vup
  end[2] = end[2] * 3                                # FLATTEN the sphere
  normalize(end)

  scroll = skytime * skyspeed
  s = truncate((scroll + 6*(64-1) * end[0]) * 65536)
  t = truncate((scroll + 6*(64-1) * end[1]) * 65536)
  RETURN (s, t)
```

**Invariants** — four decisions carry the look.

**The vertical component is tripled before normalizing**, which flattens the notional sphere into a
squashed dome. Without it the sky's texture would compress severely near the horizon; with it the horizon
looks like a distant plane. That one multiplication is the whole of the sky's projection character.

**The texture coordinate uses only the horizontal components** of the normalized direction, so the sky is a
plane projected from above — looking straight up shows the texture's origin and looking at the horizon
shows it stretched. That is exactly the artifact players remember.

**Both coordinates are offset by the same scroll value**, so the sky scrolls diagonally at a single speed.
Two layers scroll at different speeds ([`r_sky.c`](r_sky.c.md)), which is what makes it read as clouds over
a background.

**The scale factor is six times half the sky size minus one** — 378 — which maps the direction's unit range
onto the 128-texel sky texture six times over. So the sky tiles six times across the visible hemisphere.

The normalization by the larger view dimension is what keeps the sky's apparent scale constant when the
view is resized.

## `D_DrawSkyScans8`

**Contract** — fills a list of spans from the sky source, recomputing the exact texture coordinate every 32
pixels and stepping between.

```text
FUNCTION d_draw_sky_scans_8(spans)
  FOR EACH span
    pdest = the view buffer AT (span.v, span.u)
    s, t = d_sky_uv_to_st(span.u, span.v)
    LOOP over 32-pixel sub-spans, exactly as d_draw_spans_8 does, except that
      the far-end coordinate comes from d_sky_uv_to_st at the far pixel rather
      than from a gradient, and the steps are a shift by 5 or a division:
      REPEAT spancount TIMES
        dest[next] = r_skysource[((t BITAND 0x007F0000) SHIFTED RIGHT 8) +
                                 ((s BITAND 0x007F0000) SHIFTED RIGHT 16)]
        s = s + sstep ;  t = t + tstep
```

**Invariants** — the sampling masks both coordinates to **seven bits of the integer part**, so the sky
texture is 128 by 128 and wraps by masking ([`d_local.h`](d_local.h.md)). The t coordinate is shifted right
by 8 rather than 16 because that leaves it pre-multiplied by 128 — the texture's width — so the two-
dimensional index is one addition.

Unlike the world filler, there is **no clamping** of the coordinates, because the mask makes any value
legal.

The sky writes the depth buffer through a separate call in
[`d_edge.c`](d_edge.c.md), not here.
