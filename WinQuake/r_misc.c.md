# WinQuake/r_misc.c

> Per-frame setup, the frustum transform, and the renderer's own instrumentation: a timing graph, a benchmark, and four diagnostic reports.

**Needs** — [`r_local.h`](r_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`client.h`](client.h.md) · [`view.h`](view.h.md) · [`screen.h`](screen.h.md) · [`console.h`](console.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`r_main.c`](r_main.c.md) calls the frame setup and the reports
**Tier floor** — none

## Purpose

Per-frame setup for the software renderer, the frustum transform it needs, and the renderer's own instrumentation. The
instrumentation is the interesting half: a timing graph, a rotation benchmark and four textual reports, which together are
the only performance visibility the engine has.

## State

```text
VARIABLE the frame counter and the per-frame counters the reports print
VARIABLE the timing graph's history ring
```

## `R_SetupFrame`

**Contract** — per frame: validates the tunables, advances the frame counter, records the view position and
basis from the view description, finds the leaf the view is in, decides whether the underwater warp applies,
transforms the frustum into view space, resets the pools and counters, and reinitializes the surface cache if
the resolution changed.

```text
FUNCTION r_setup_frame()
  r_check_variables()
  r_animate_light() ;  r_framecount = r_framecount + 1
  r_origin = r_refdef.vieworg
  vpn, vright, vup = the basis OF r_refdef.viewangles
  r_viewleaf = the leaf containing r_origin
  # The underwater warp applies when the view is in a liquid AND the tunable
  # allows it.
  r_dowarp = (r_viewleaf's contents is a liquid) AND the warp tunable
  IF r_dowarp differs from last frame OR the view changed
    reconfigure the whole renderer and the surface cache
  v_setcontentscolor(r_viewleaf.contents)
  r_transform_frustum()
  r_set_up_frustum_indexes()
  reset the edge and surface pools, the counters and the sub-model list
```

**Invariants** — **entering or leaving water reconfigures the renderer**, because the destination buffer and its
stride change ([`d_init.c`](d_init.c.md#d_setupframe)). That is why the transition is visibly not free.

The view leaf is found before anything else because three decisions depend on it: the warp, the screen tint,
and the visibility marking.

## `R_TransformFrustum`, `TransformVector`, `R_TransformPlane`

**Contract** — `TransformVector` takes a world vector into view space by three dot products with the basis.
`R_TransformFrustum` transforms the four screen-edge planes into world space, producing the clip planes the
tree walk and the edge clipper test against. `R_TransformPlane` transforms one plane.

**Invariants** — the frustum is defined in *view* space
([`r_main.c`](r_main.c.md#r_viewchanged)) and transformed into *world* space here, because the tree walk tests
world-space boxes. The source notes that a rotating entity would need the view matrix instead — which is why
drawing a rotated sub-model re-derives the frustum ([`r_bsp.c`](r_bsp.c.md#r_entityrotate-r_rotatebmodel)).

## `R_SetUpFrustumIndexes`

**Contract** — for each clip plane, precomputes the six array indices that select the box corner nearest and
furthest from that plane, from the plane's component signs.

**Invariants** — this is what makes the tree walk's cull three indexed loads rather than three comparisons
([`r_bsp.c`](r_bsp.c.md#r_recursiveworldnode)). Computed once per view change.

## `R_CheckVariables`

**Contract** — notices a change to the full-bright or ambient tunables and flushes the surface cache, because
every cached surface was built with the old values.

## `R_TimeRefresh_f`

**Contract** — the `timerefresh` command; renders 128 frames while rotating the view a full turn, and reports
the elapsed time and the resulting frame rate. The engine's own benchmark.

**Invariants** — this is the benchmark the recipe's conformance section names
([`SYSTEM-REQUIREMENTS.md`](../SYSTEM-REQUIREMENTS.md#6-conformance)) and the natural thing for a rebuild to
compare against.

## `R_LineGraph`, `R_TimeGraph`

**Contract** — draw a scrolling bar graph of the last hundred frames' durations directly into the surface,
one column per frame at a configurable height.

**Invariants** — a per-frame profiler visible in the game, which is how the renderer's cost was understood at
all. The source notes it should be disabled on backends with no readable buffer.

## `R_PrintTimes`, `R_PrintDSpeeds`, `R_PrintAliasStats`

**Contract** — print the frame's pass timings ([`r_local.h`](r_local.h.md) names the twelve variables), the
rasterizer's per-pass breakdown, and the model pass's polygon counts.

## `WarpPalette`

**Contract** — builds a tinted palette for the underwater effect.

**Notes** — superseded by the colour-shift compositing in [`view.c`](view.c.md); not called.
