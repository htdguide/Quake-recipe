# WinQuake/d_polyse.c

> The animated-model rasterizer: an affine triangle filler with gouraud shading and a depth test, a recursive-subdivision alternative for near models, and a twelve-case edge table that classifies a triangle's shape.

**Needs** — [`d_local.h`](d_local.h.md) · [`r_local.h`](r_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [`adivtab.h`](adivtab.h.md) · [`mathlib.h`](mathlib.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops) ([`d_polysa.s`](d_polysa.s.md))
**Used by** — [`r_alias.c`](r_alias.c.md) fills the mesh descriptor and calls
**Tier floor** — none

## Purpose

Models are rasterized differently from the world, and this file is why. A model triangle is small, so
**affine** texture mapping — linear in screen space, no perspective division — is close enough; and a model
is lit per vertex, so the light value is interpolated alongside the texture coordinate. The world's
rasterizer does neither.

There are two paths. The ordinary one fills a triangle by scanning its edges. The other **recursively
subdivides** a triangle until its edges are one pixel long and plots the midpoints as points, which is used
for models near the camera where affine mapping would visibly distort. Which path runs is decided per model
by the transition distance in [`r_local.h`](r_local.h.md).

## State

```text
CONSTANT dps_maxspans = 1025            # one per scanline plus a terminator

RECORD SpanPackage               # one scanline's worth of interpolator state
  pdest : bytes                  # where to write
  pz    : depth pointer
  count : int
  ptex  : bytes                  # the WHOLE-texel texture address
  sfrac, tfrac : int             # the FRACTIONAL texture position
  light : int                    # 8.8 fixed point
  zi    : int

RECORD EdgeTable                 # how to walk a triangle of one shape
  isflattop : int
  numleftedges  : int ;  pleftedgevert0..2  : pointers to vertices
  numrightedges : int ;  prightedgevert0..2 : pointers to vertices

VARIABLE edgetables : EdgeTable[12]      # the twelve vertex orderings
VARIABLE r_p0, r_p1, r_p2 : int[6]       # the triangle's three vertices
VARIABLE a_spans : SpanPackage           # the per-scanline packages
VARIABLE skintable : bytes[480]          # a row address per skin row
VARIABLE d_xdenom : int                  # twice the triangle's signed area
VARIABLE the gradient set: r_sstepx/y, r_tstepx/y, r_lstepx/y, r_zistepx/y,
         a_sstepxfrac, a_tstepxfrac, a_ststepxwhole
VARIABLE the per-scanline step set: d_ptexbasestep/extrastep, d_sfracbasestep,
         d_pdestbasestep, d_lightbasestep, d_zibasestep, and their extra
         counterparts
```

**Invariants** — the texture position is split into a **whole-texel address and a fraction**, so the inner
loop advances the address by a precomputed whole step and adds the fraction's carry. That split is why
there is no multiply per pixel.

The twelve edge tables are the twelve ways three vertices can be ordered by y with ties, and each says which
vertices form the left chain and which the right. That classification replaces a general polygon scan with
a table lookup ([`D_PolysetSetEdgeTable`](#d_polysetsetedgetable)).

## `D_PolysetDraw`

**Contract** — allocates the span-package array on the stack, cache-aligned, and dispatches to the
subdividing or the scanning path according to the mesh descriptor's draw type.

**Invariants** — the array is one entry per scanline plus a terminator plus one extra, with the comment
noting the extra is for cache-line pretouching. About 33 kilobytes of stack per call.

## The scanning path

### `D_DrawNonSubdiv`

**Contract** — for each triangle: computes twice its signed area, skips it if not front-facing, copies its
three vertices into the working globals, applies the seam adjustment to back-facing triangles, classifies
its shape, and rasterizes it.

```text
FUNCTION d_draw_non_subdiv()
  FOR EACH triangle
    i0, i1, i2 = its three final vertices
    d_xdenom = (i0.v - i1.v)*(i0.u - i2.u) - (i0.u - i1.u)*(i0.v - i2.v)
    IF d_xdenom >= 0  CONTINUE                   # BACKFACE CULL, and also
                                                 # rejects degenerate triangles
    copy all six components of each vertex INTO r_p0, r_p1, r_p2
    IF the triangle does NOT face front
      FOR EACH vertex flagged as on the seam
        add the seam fix-up TO its s coordinate  # modelgen.h's seam rule
    d_polyset_set_edge_table()
    d_rasterize_alias_poly_smooth()
```

**Invariants** — **the signed area serves as both the backface test and the gradient denominator**, which is
why it is computed once and kept. A non-negative value means the triangle faces away *or* has zero area, and
both are correctly skipped by one test.

**The seam adjustment is applied to the working copy, not the vertex**, so a vertex shared between a
front-facing and a back-facing triangle gets the right coordinate in each. The subdividing path instead
modifies the vertex and restores it afterwards — a different solution to the same problem, in the same file.

### `D_PolysetCalcGradients`

**Contract** — computes the per-pixel and per-scanline steps for texture s, texture t, light and depth, from
the three vertices and the signed area.

```text
FUNCTION d_polyset_calc_gradients(skinwidth)
  # The standard plane-equation gradients over a triangle.
  xstepdenominv =  1 / d_xdenom
  ystepdenominv = -xstepdenominv            # screen y is inverted

  FOR EACH interpolated quantity q IN { light, s, t, depth reciprocal }
    t0 = r_p0[q] - r_p2[q] ;  t1 = r_p1[q] - r_p2[q]
    stepx = (t1*(p0.v - p2.v) - t0*(p1.v - p2.v)) * xstepdenominv
    stepy = (t1*(p0.u - p2.u) - t0*(p1.u - p2.u)) * ystepdenominv
    # ...with ONE exception: the LIGHT steps are rounded UP, not truncated.
    IF q IS light  stepx = ceil(stepx) ;  stepy = ceil(stepy)

  # Split the s and t steps into whole texels and fractions.
  a_sstepxfrac = r_sstepx BITAND 0xFFFF
  a_tstepxfrac = r_tstepx BITAND 0xFFFF
  a_ststepxwhole = skinwidth * (r_tstepx SHIFTED RIGHT 16)
                            + (r_sstepx SHIFTED RIGHT 16)
```

**Invariants** — three decisions.

**The light steps are rounded up rather than truncated**, and the comment gives the reason precisely:
rounding up exaggerates positive steps and diminishes negative ones, biasing away from underflow toward
overflow — because **underflow is very visible and overflow is very unlikely**, the latter because ambient
lighting keeps the values away from the top of the range. Underflow would make a lit vertex go black
mid-triangle. That is a considered trade and a rebuild should reproduce it.

**The whole-texel step is precomputed as one address delta**, combining the t step's whole part times the
skin width with the s step's whole part. So advancing one pixel is one addition to the address plus the two
fraction carries.

**The vertical gradients are negated** because screen y increases downward.

### `D_PolysetSetUpForLineScan`

**Contract** — takes two fixed-point endpoints of an edge; computes a Bresenham-style base step and error
terms. Uses the precomputed division table when both deltas are small, and the floor-based division
otherwise.

```text
FUNCTION d_polyset_set_up_for_line_scan(u0, v0, u1, v1)
  errorterm = -1
  tm = u1 - u0 ;  tn = v1 - v0
  IF both tm and tn lie IN -15 .. 16
    (ubasestep, erroradjustup) = adivtab[(tm+15)*32 + (tn+15)]    # a LOOKUP
  ELSE
    (ubasestep, erroradjustup) = floor_div_mod(tm, tn)            # a division
  erroradjustdown = tn
```

**Invariants** — the table ([`adivtab.h`](adivtab.h.md)) covers the common case and the general division is
the fallback; both must produce **floor-based** quotients and remainders
([`mathlib.c`](mathlib.c.md#floordivmod)), because the error term's sign convention depends on it. A
rebuild should just divide.

### `D_PolysetScanLeftEdge`

**Contract** — walks a triangle's left edge down a given number of scanlines, emitting one span package per
scanline with every interpolator's current value, and stepping each by either its base step or its extra
step according to the Bresenham error term.

```text
FUNCTION d_polyset_scan_left_edge(height)
  REPEAT height TIMES
    emit a span package holding d_pdest, d_pz, d_aspancount, d_ptex,
         d_sfrac, d_tfrac, d_light, d_zi
    errorterm = errorterm + erroradjustup
    IF errorterm >= 0                          # the edge stepped an extra pixel
      advance every interpolator by its EXTRA step
      errorterm = errorterm - erroradjustdown
    ELSE
      advance every interpolator by its BASE step
    # In both cases, carry the texture fractions into the address:
    d_ptex = d_ptex + (d_sfrac SHIFTED RIGHT 16) ;  d_sfrac = low 16 bits
    IF d_tfrac carried  d_ptex = d_ptex + skinwidth ;  d_tfrac = low 16 bits
```

**Invariants** — **eight interpolators advance in lockstep**, each with a base and an extra step, and the
Bresenham error term selects between them. That is the standard scanline setup, and its cost is why the span
package exists at all: the state is captured once per scanline and the fill loop re-reads it rather than
recomputing.

The source's own note asks whether the light, s and t values need clamping at both ends. They do not in
practice because the vertices were clamped, but a rebuild should check.

### `D_PolysetDrawSpans8`

**Contract** — takes the span-package array; for each package, fills that scanline's run with textured,
shaded, depth-tested pixels. The array is terminated by a sentinel count.

```text
FUNCTION d_polyset_draw_spans_8(packages)
  FOR EACH package UNTIL the sentinel
    # The RIGHT edge's Bresenham runs here, giving this scanline's length.
    lcount = d_aspancount - package.count
    errorterm = errorterm + erroradjustup
    IF errorterm >= 0  d_aspancount = d_aspancount + d_countextrastep
                       errorterm = errorterm - erroradjustdown
    ELSE               d_aspancount = d_aspancount + ubasestep

    IF lcount != 0
      load the eight interpolators FROM the package
      REPEAT lcount TIMES
        IF (zi SHIFTED RIGHT 16) >= the stored depth
          dest = colormap[texel + (light BITAND 0xFF00)]
          store the depth
        advance dest and the depth pointer
        zi = zi + r_zistepx ;  light = light + r_lstepx
        ptex = ptex + a_ststepxwhole
        sfrac = sfrac + a_sstepxfrac ;  carry it into ptex
        tfrac = tfrac + a_tstepxfrac ;  IF it carried, add skinwidth to ptex
```

**Invariants** — four things.

**The depth test is greater-or-equal**, so a model's later triangle wins a tie against an earlier one at the
same depth. That keeps adjacent triangles of one model coherent.

**Lighting is one indexed load**: the light's high byte selects a shading-table row and the texel selects a
column, exactly as the world's surface builder does
([`vid.h`](vid.h.md#the-lighting-table)). So gouraud shading costs nothing beyond the interpolation.

**The left edge's scan and the right edge's Bresenham are split between two functions**, with the left one
emitting packages and this one consuming them while running the right edge. That split exists so the left
edge's eight-interpolator walk can be one tight loop.

**No transparency test.** A model's skin has no transparent texels, unlike a sprite's.

### `D_PolysetFillSpans8`

**Contract** — fills the packages with a single incrementing colour, for the flat-shading diagnostic. Does no
depth testing, which the source marks as missing.

### `D_PolysetSetEdgeTable`

**Contract** — classifies the triangle's three vertices by their vertical ordering and selects one of twelve
precomputed descriptions of which vertices form the left chain and which the right.

**Invariants** — twelve cases rather than six because a flat top or a flat bottom is distinguished from a
general ordering; the table's `isflattop` field records it. Selecting from a table rather than sorting is
what keeps the setup to a few comparisons.

### `D_RasterizeAliasPolySmooth`

**Contract** — using the selected edge table, sets up the left and right edge scans and the gradients, then
alternates between scanning the left edge and filling, once per chain segment.

**Invariants** — the triangle is rasterized in **one or two vertical segments** depending on whether the
edge chains change vertex at the same scanline, which is what the edge table's left- and right-edge counts
say. That is the general two-part triangle scan.

## The subdividing path

### `D_DrawSubdiv`

**Contract** — for each front-facing triangle, selects the shading-table row from the *first* vertex's light
value and recursively subdivides. Applies the seam adjustment by modifying the vertices and restoring them
afterwards.

**Invariants** — **the whole triangle uses one light level**, taken from vertex zero, because the recursive
path does not interpolate light. So a near model is flat-shaded per triangle while a distant one is gouraud
shaded — the opposite of what one would expect, and it is invisible because near triangles are small on
screen.

### `D_PolysetRecursiveTriangle`

**Contract** — takes three vertices; if every edge is at most one pixel long in both axes, returns, the
triangle being considered filled. Otherwise splits the longest offending edge at its midpoint, plots the
midpoint if the split edge is a leading edge, and recurses on the two halves.

```text
FUNCTION d_polyset_recursive_triangle(p1, p2, p3)
  # Find an edge longer than one pixel; rotate the vertices so it is p1-p2.
  IF |p2.u - p1.u| > 1 OR |p2.v - p1.v| > 1  split p1-p2
  IF |p3.u - p2.u| > 1 OR |p3.v - p2.v| > 1  rotate so it is p1-p2, then split
  IF |p1.u - p3.u| > 1 OR |p1.v - p3.v| > 1  rotate again, then split
  RETURN                                     # every edge is within one pixel:
                                             # the triangle is FILLED

split:
  new = the componentwise midpoint OF p1 and p2      # u, v, s, t and depth;
                                                    # NOT light
  # Plot the midpoint only when splitting a LEADING edge, so that each pixel
  # is plotted by exactly one of the triangles sharing it.
  IF p2.v > p1.v                             GOTO nodraw
  IF p2.v == p1.v AND p2.u < p1.u            GOTO nodraw
  IF (new.zi SHIFTED RIGHT 16) >= the stored depth
    store it ;  store d_pcolormap[skintable[new.t][new.s]]
nodraw:
  d_polyset_recursive_triangle(p3, p1, new)
  d_polyset_recursive_triangle(p3, new, p2)
```

**Invariants** — three things, and the second is the clever one.

**A triangle whose every edge spans at most one pixel is considered filled** and nothing is drawn for it.
So the triangle's interior is covered entirely by the *midpoints plotted during subdivision*, not by a fill.
That is why this is a plotting path rather than a filling one.

**The leading-edge test is what makes each pixel plotted exactly once.** Two adjacent triangles share an
edge and would both plot its midpoints; the test — the edge must go upward, or horizontally leftward —
selects exactly one of the two orientations, so only one triangle plots it. Without the test the model is
drawn twice over and the depth test does not save it, because the depths are equal.

**The light is not interpolated** — the midpoint's light component is not computed — which is why the
caller fixes a single shading row per triangle.

### `D_PolysetDrawFinalVerts`

**Contract** — takes a run of final vertices; plots each as a single depth-tested pixel, skipping any at or
beyond the view's right or bottom edge.

**Invariants** — the exclusion of the right and bottom edges is explained by the comment: those coordinates
are legal for *filling* under the fill rule but must not be *drawn*, because the fill rule treats them as
exclusive. Plotting them would write one pixel outside the view.

This is the entry point the renderer uses when a model is far enough away that individual vertices are
adequate ([`r_alias.c`](r_alias.c.md)), which is the third and coarsest of the three model paths.

## `D_PolysetUpdateTables`

**Contract** — rebuilds the per-row skin address table when the skin changes.

**Invariants** — a table of row addresses rather than a multiply per texel fetch, the same optimization the
scanline tables in [`d_modech.c`](d_modech.c.md) apply to the destination.

**Notes** — the file contains a disabled palette-remapping experiment, and two disabled alternative
recursive routines. None ships.
