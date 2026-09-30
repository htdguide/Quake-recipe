# WinQuake/d_scan.c

> The span fillers: perspective-correct texture mapping by dividing once every eight pixels, the liquid sine warp, the depth-only pass, and the underwater screen warp.

**Needs** — [`d_local.h`](d_local.h.md) · [`r_local.h`](r_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`client.h`](client.h.md) (the clock, for the warp phase) · [`vid.h`](vid.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops) (three routines have twins in [`d_draw.s`](d_draw.s.md), [`d_draw16.s`](d_draw16.s.md) and [`d_scana.s`](d_scana.s.md))
**Used by** — [`d_edge.c`](d_edge.c.md) dispatches to these per surface; [`screen.c`](screen.c.md) calls the screen warp
**Tier floor** — none; the arithmetic is tier-free, though the fixed-point formats must be reproduced exactly

## Purpose

The file where pixels are actually written for the world. Its whole content is one idea applied three
ways: **divide rarely, interpolate in between**.

Perspective-correct texture mapping needs a division per pixel. This engine performs one division per
**eight** pixels and steps linearly between, choosing eight because it makes the step a shift rather than
a division. The resulting error is the faint sawtooth visible on surfaces at grazing angles, and it is the
renderer's most conspicuous approximation.

## State

```text
VARIABLE r_turb_pbase, r_turb_pdest : bytes         # the liquid warp's source
                                                    # and destination
VARIABLE r_turb_s, r_turb_t : fixed16               # its current texture
                                                    # coordinate
VARIABLE r_turb_sstep, r_turb_tstep : fixed16
VARIABLE r_turb_turb : list<int>                    # its sine table, phase-shifted
VARIABLE r_turb_spancount : int
```

**Invariants** — the liquid filler communicates entirely through globals, which is what lets its inner
loop be replaced by assembly with no argument marshalling
([`quakeasm.h`](quakeasm.h.md#the-shared-globals)).

## `D_DrawSpans8`

**Contract** — takes a list of spans and fills each from the cached surface, sampling with
perspective-corrected texture coordinates. Reads the nine span-gradient globals, the surface's texture
offset and its clamping bounds, all set by [`d_edge.c`](d_edge.c.md) before the call.

```text
FUNCTION d_draw_spans_8(spans)
  pbase = the cached surface
  # Precompute the gradient's advance over eight pixels.
  sdivz8stepu = d_sdivzstepu * 8
  tdivz8stepu = d_tdivzstepu * 8
  zi8stepu    = d_zistepu * 8

  FOR EACH span
    pdest = the view buffer AT (span.v, span.u)
    count = span.count

    # --- the exact texture coordinate at the span's START ---
    sdivz = d_sdivzorigin + span.v*d_sdivzstepv + span.u*d_sdivzstepu
    tdivz = d_tdivzorigin + span.v*d_tdivzstepv + span.u*d_tdivzstepu
    zi    = d_ziorigin    + span.v*d_zistepv    + span.u*d_zistepu
    z = 65536.0 / zi                       # ONE division, prescaled to 16.16
    s = truncate(sdivz * z) + sadjust ;  clamp INTO 0 .. bbextents
    t = truncate(tdivz * z) + tadjust ;  clamp INTO 0 .. bbextentt

    LOOP
      spancount = min(count, 8)
      count = count - spancount

      IF count != 0                         # a full eight-pixel sub-span
        # Advance the gradient eight pixels and divide again.
        sdivz = sdivz + sdivz8stepu ;  tdivz = tdivz + tdivz8stepu
        zi = zi + zi8stepu
        z = 65536.0 / zi
        snext = truncate(sdivz * z) + sadjust
        clamp snext INTO 8 .. bbextents     # NOTE the LOW bound of 8, not 0
        tnext = truncate(tdivz * z) + tadjust
        clamp tnext INTO 8 .. bbextentt
        sstep = (snext - s) SHIFTED RIGHT 3    # a shift, because the interval
        tstep = (tnext - t) SHIFTED RIGHT 3    # is exactly 8
      ELSE                                  # the final, short sub-span
        # Advance to the LAST PIXEL rather than past it, so the interpolation
        # cannot step off the polygon; then divide to get the step.
        n = spancount - 1
        sdivz = sdivz + d_sdivzstepu * n ;  tdivz = tdivz + d_tdivzstepu * n
        zi = zi + d_zistepu * n
        z = 65536.0 / zi
        snext = truncate(sdivz * z) + sadjust ;  clamp INTO 8 .. bbextents
        tnext = truncate(tdivz * z) + tadjust ;  clamp INTO 8 .. bbextentt
        IF spancount > 1
          sstep = (snext - s) / (spancount - 1)      # a real division
          tstep = (tnext - t) / (spancount - 1)

      REPEAT spancount TIMES                # the inner loop
        dest[next] = pbase[(s SHIFTED RIGHT 16) +
                           (t SHIFTED RIGHT 16) * cachewidth]
        s = s + sstep ;  t = t + tstep

      s = snext ;  t = tnext                # resume exactly, not from the
                                            # accumulated value
```

**Invariants** — six details, and every one is load-bearing.

**Eight pixels per division**, and the step is therefore a right shift by three. Sixteen would halve the
divisions and double the error; four would do the reverse. Eight is the tuned choice and it is what every
published screenshot of this engine shows.

**The division is `65536 / zi`, prescaled**, so that multiplying by it yields a 16.16 fixed-point texture
coordinate directly with no second scaling.

**The sub-span's endpoint is computed exactly and the coordinate resumes from it**, not from the
accumulated interpolation. So error does not accumulate across a long span — each eight-pixel run is
independently anchored. A rebuild that simply keeps stepping drifts visibly over a wide wall.

**The low clamp is 8, not 0.** The comment says why: the step can be negative, and a coordinate computed
as slightly below zero would, after eight negative steps, index before the texture. Clamping the
*endpoint* to 8 guarantees the whole run stays inside. That is a subtle and necessary guard, and it is why
the low bound differs between the initial clamp (0) and the per-sub-span clamp (8).

**The final sub-span advances to its last pixel rather than one past it**, and then divides by
`spancount - 1`. Advancing past would sample outside the polygon; dividing by the count rather than the
count minus one would land short. The comment calls this biasing the steps low.

**The upper clamps are the surface's own extents**, which is why liquid surfaces have theirs set
enormous ([`model.c`](model.c.md#mod_loadfaces)) — their warp deliberately samples outside.

## `D_DrawSpans16`

**Contract** — identical, with each sampled index expanded through the sixteen-bit shading table and the
destination advancing two bytes per pixel. Not present in this file; the portable version lives in the
same routine under a compile-time switch and the accelerated one in
[`d_draw16.s`](d_draw16.s.md).

## `Turbulent8`

**Contract** — fills a list of spans from a 64-by-64 liquid texture, displacing each texture coordinate by
a sine of the *other* coordinate and the time. Uses sixteen-pixel sub-spans rather than eight.

```text
FUNCTION turbulent_8(spans)
  # Phase-shift the sine table by the clock, so the ripple scrolls.
  r_turb_turb = sintable + (truncate(cl.time * 20) BITAND 127)
  r_turb_pbase = the cached surface
  ...exactly the structure of d_draw_spans_8, with 16-pixel sub-spans and
     the low clamps at 16 rather than 8, and then per sub-span:
  r_turb_s = r_turb_s BITAND ((128 << 16) - 1)      # wrap within the cycle
  r_turb_t = r_turb_t BITAND ((128 << 16) - 1)
  d_draw_turbulent_8_span()
```

And the inner loop:

```text
FUNCTION d_draw_turbulent_8_span()
  REPEAT r_turb_spancount TIMES
    # Each coordinate is displaced by a sine indexed by the OTHER one.
    sturb = ((r_turb_s + r_turb_turb[(r_turb_t SHIFTED RIGHT 16) BITAND 127])
             SHIFTED RIGHT 16) BITAND 63
    tturb = ((r_turb_t + r_turb_turb[(r_turb_s SHIFTED RIGHT 16) BITAND 127])
             SHIFTED RIGHT 16) BITAND 63
    dest[next] = pbase[(tturb SHIFTED LEFT 6) + sturb]
    r_turb_s = r_turb_s + r_turb_sstep
    r_turb_t = r_turb_t + r_turb_tstep
```

**Invariants** — **each coordinate is displaced by a sine of the other**, which is what makes the
distortion a two-dimensional swirl rather than a one-dimensional shear. That cross-coupling is the whole
visual character of water in this game.

The texture is 64 by 64 so the final wrap is a mask with 63; the sine table's cycle is 128 so its index is
a mask with 127; and the coordinate wrap before the loop is a mask against the cycle scaled into 16.16.
Three masks, no modulos ([`d_iface.h`](d_iface.h.md)).

**Sixteen-pixel sub-spans rather than eight**, because a liquid surface has no lightmap and its
perspective error is hidden by the distortion — so the cheaper subdivision is affordable.

The sine table is **phase-shifted by indexing into it** rather than by adding to the coordinate, which
costs nothing per pixel.

The scroll speed of 20 and the amplitude are in [`r_local.h`](r_local.h.md), and the same numbers appear
in the hardware renderer's table ([`gl_warp_sin.h`](gl_warp_sin.h.md)).

## `D_DrawZSpans`

**Contract** — writes only the depth buffer for a list of spans, interpolating one over depth linearly
with no division at all.

```text
FUNCTION d_draw_z_spans(spans)
  # The step, scaled into the depth buffer's 16-bit representation.
  izistep = truncate(d_zistepu * 0x8000 * 0x10000)
  FOR EACH span
    pdest = the depth buffer AT (span.v, span.u)
    zi = d_ziorigin + span.v*d_zistepv + span.u*d_zistepu
    izi = truncate(zi * 0x8000 * 0x10000)
    # Align to an even address, then write two 16-bit values per 32-bit store.
    IF pdest is odd-aligned  write one value ;  advance ;  count = count - 1
    REPEAT count/2 TIMES
      pack two successive (izi >> 16) values into one 32-bit word and store it
    IF count is odd  write the final value
```

**Invariants** — **no division**, because one over depth is linear in screen space. This is the cheapest
loop in the renderer and it is why the world pass can seed a depth buffer for the model and sprite passes
nearly for free ([`d_local.h`](d_local.h.md#the-depth-buffer)).

The scaling by `0x8000 * 0x10000` is a 31-bit shift expressed as two multiplications, and the source's
own comment admits it **relies on floating-point exceptions being masked** to avoid a range problem: for
a very near surface the product overflows, and the result is whatever the conversion produces. That is
why [`sys.h`](sys.h.md#floating-point-control) exists. A rebuild should clamp the depth explicitly; the
comment marks the range check as unfinished.

The two-at-a-time packing is an alignment optimization. Incidental.

## `D_WarpScreen`

**Contract** — resamples the rendered frame from the warp buffer to the display through a
time-varying sine displacement in both axes, compressing slightly so the displaced edges do not wrap.
Called when the player is underwater.

```text
FUNCTION d_warp_screen()
  # Precompute a source row address per destination row and a source column
  # per destination column, INCLUDING the compression: the source is sampled
  # over a range scaled by h/(h + 2*amplitude), so the displacement never
  # reaches outside it.
  FOR EACH destination row v IN 0 .. height + 2*amp2 - 1
    rowptr[v] = the warp buffer's row AT v * hratio * h / (h + 2*amp2)
  FOR EACH destination column u IN 0 .. width + 2*amp2 - 1
    column[u] = the warp buffer's column AT u * wratio * w / (w + 2*amp2)

  turb = intsintable + (truncate(cl.time * 20) BITAND 127)   # phase by the clock
  FOR EACH destination row v
    # The ROW's displacement is a sine of v; the COLUMN's is a sine of u.
    col = &column[turb[v]] ;  row = &rowptr[v]
    FOR EACH destination column u
      dest[u] = row[turb[u]][col[u]]
```

**Invariants** — **the compression is the reason the two precomputed tables exist.** Displacing a sample
by up to the amplitude would read outside the frame at the edges; so the source is sampled over a
slightly smaller region and the displacement fits inside it. The comment states exactly this. A rebuild
that omits the compression gets wrapped garbage along all four edges.

The displacement is **separable**: the row index is shifted by a sine of the row and the column index by a
sine of the column, both read from the same phase-shifted table. That is what makes the inner loop two
indexed loads.

The warp buffer is a fixed 320 by 200 regardless of display resolution
([`d_iface.h`](d_iface.h.md)), so the effect is blockier at higher resolutions — a visible and
well-known artifact.

Two stack arrays sized to the maximum display dimensions plus the amplitude are allocated per call, about
9 kilobytes. A rebuild allocates them once.
