# WinQuake/gl_warp_sin.h

> A table of 320 sine values scaled for the hardware renderer's water warp.

**Needs** — nothing; a table of literals
**Used by** — [`gl_warp.c`](gl_warp.c.md)
**Tier floor** — none

## Purpose

The hardware renderer subdivides a liquid surface into a grid and displaces each vertex's texture
coordinate by a sine of its position and the time. This is that sine, tabulated.

## State

```text
CONSTANT turbsin : real[320]
    # sin over one full period, scaled so the values run 0 .. 8 .. 0 .. -8 .. 0
```

**Invariants** — 320 entries over one period, and the amplitude is 8 — matching the software
renderer's warp amplitude ([`r_local.h`](r_local.h.md)), so the two renderers' water ripples at the
same strength. A rebuild changing one must change the other or the same map looks different in the two
builds.

The period of 320 is what makes the index a masked addition rather than a modulo when combined with the
surface's texture coordinates.

**Notes** — the software renderer builds its equivalent tables at startup
([`r_main.c`](r_main.c.md)) rather than tabulating them in the source. There is no reason for the
asymmetry, and a rebuild should compute both.

```text
FUNCTION build_turbsin() -> real[320]
  FOR EACH i IN 0..319
    turbsin[i] = 8.0 * sin(i * 2*pi / 320)
```

Checking that generator against the file's first few values — 0, 0.19633, 0.392541 — confirms it.
