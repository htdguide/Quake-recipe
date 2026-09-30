# WinQuake/d_modech.c

> Recomputes everything the rasterizer derives from the display's shape, whenever the view or the mode changes — including the particle size scaling and the per-scanline address tables.

**Needs** — [`d_local.h`](d_local.h.md) · [`r_local.h`](r_local.h.md) · [`vid.h`](vid.h.md)
**Used by** — [`r_main.c`](r_main.c.md) calls it from the view-changed path
**Tier floor** — none

## Purpose

Everything the rasterizer derives from the display's shape, recomputed whenever the view rectangle or the display mode
changes. It is the one place those derivations live, which is why a mode change is a single call.

## State

```text
VARIABLE d_vrectx, d_vrecty : int                        # the view's origin
VARIABLE d_vrectright_particle, d_vrectbottom_particle : int
VARIABLE d_y_aspect_shift, d_pix_min, d_pix_max, d_pix_shift : int
VARIABLE d_scantable  : int[1024]                        # byte offset per
                                                         # scanline
VARIABLE zspantable   : pointer[1024]                    # depth row address per
                                                         # scanline
```

## `D_ViewChanged`

**Contract** — recomputes the mip scale reference, the depth buffer's geometry, the particle size range,
the aspect-ratio shift, the particle clipping bounds, and both per-scanline address tables. Then patches
the self-modifying rasterizer.

```text
FUNCTION d_view_changed()
  rowbytes = 320 IF the warp is active ELSE the surface's stride
  scale_for_mip = max(xscale, yscale)         # mip selection uses the LARGER
  d_zrowbytes = vid.width * 2 ;  d_zwidth = vid.width

  # --- particle sizes, scaled from the 320-wide reference resolution ---
  d_pix_min = max(view width / 320, 1)
  d_pix_max = round(view width / 80)          # i.e. 320/4
  d_pix_shift = 8 - round(view width / 320)
  IF d_pix_max < 1  d_pix_max = 1

  d_y_aspect_shift = 1 IF pixelAspect > 1.4 ELSE 0

  d_vrectx = the view's x ;  d_vrecty = the view's y
  d_vrectright_particle  = the view's right  - d_pix_max
  d_vrectbottom_particle = the view's bottom - (d_pix_max SHIFTED LEFT
                                                d_y_aspect_shift)

  FOR EACH scanline i IN 0 .. vid.height-1
    d_scantable[i] = i * rowbytes
    zspantable[i]  = the depth buffer + i * d_zwidth
  d_patch()
```

**Invariants** — four things.

**The mip reference is the larger of the two projection scales**, so a non-square pixel aspect does not
make one axis mip earlier than the other.

**Particle sizes are scaled from a 320-wide reference.** A particle is a square of between `d_pix_min` and
`d_pix_max` pixels, and at 320 wide those are 1 and 4. So particles keep their apparent size across
resolutions rather than shrinking — which is the right choice and is why the numbers are ratios rather
than constants.

**The aspect shift doubles a particle's height** on a display whose pixels are much taller than wide, which
the base VGA mode's are. One comparison against 1.4 selects it.

**The particle clipping bounds are inset by the maximum particle size**, so the plotter needs no per-pixel
bounds check — a particle whose centre passes the test cannot write outside the view. That is the
optimization the whole set of variables exists for.

The two address tables let a rasterizer reach scanline *v* with one indexed load. They are rebuilt on every
view change because the stride changes with the warp.

## `D_Patch`

**Contract** — on the accelerated build, makes the model rasterizer's self-modifying region writable, once.
Otherwise does nothing.

**Invariants** — the pairing with [`sys.h`](sys.h.md#sys_makecodewriteable) and the reason it exists are in
[`d_polysa.s`](d_polysa.s.md). A rebuild that passes the value in a register deletes this.
