# WinQuake/r_sky.c

> Builds the scrolling sky texture by compositing two layers of a single 256-by-128 source, and regenerates it when the scroll has moved far enough.

**Needs** — [`r_local.h`](r_local.h.md) · [`d_local.h`](d_local.h.md) · [`model.h`](model.h.md) · [`client.h`](client.h.md)
**Used by** — [`model.c`](model.c.md#mod_loadtextures) calls the initializer on a sky texture; [`d_edge.c`](d_edge.c.md) triggers regeneration; [`d_sky.c`](d_sky.c.md) samples the result
**Tier floor** — none

## Purpose

The sky is two layers scrolling at different speeds, and the source art holds both side by side in one
texture. This file splits them and composites them into a single 128-by-128 tile that
[`d_sky.c`](d_sky.c.md) then samples, regenerating that tile whenever the scroll offset changes by a whole
texel.

Compositing once per few frames rather than blending per pixel is the whole idea: the sky costs one texture
fetch per pixel, like a wall.

## State

```text
CONSTANT skyshift = 7 ;  skysize = 128 ;  skymask = 127
VARIABLE bottomsky, bottommask : bytes       # the two split layers
VARIABLE newsky : bytes                      # the composited tile
VARIABLE r_skysource : bytes                 # what d_sky.c samples
VARIABLE skytime, skyspeed, skyspeed2 : real
VARIABLE r_skymade : int                     # whether the tile is current
VARIABLE r_skydirect : int                   # unused
```

**Invariants** — the two scroll speeds differ, which is what makes the sky read as clouds moving over a
background rather than as one sliding image. Their ratio is fixed here.

## `R_InitSky`

**Contract** — takes a loaded texture whose name begins with the sky prefix; splits its 256-by-128 pixels into
two 128-by-128 layers — the left half is the foreground with palette index 0 meaning transparent, the right
half the background — and records them.

**Invariants** — **the source art is 256 by 128 and holds both layers side by side.** That is a convention of
the asset pipeline, not of the file format, and it is why the sky texture is twice as wide as it is tall. A
rebuild must split the same way.

The foreground layer's transparency is **palette index 0**, not 255 as everywhere else in the engine. A real
inconsistency; a rebuild must use 0 here.

## `R_MakeSky`

**Contract** — composites the two layers into the tile at the current scroll offsets: for each texel, the
foreground layer's texel at one offset if it is not transparent, otherwise the background's at the other
offset. Records the time it was built.

```text
FUNCTION r_make_sky()
  offset1 = the foreground's scroll, in whole texels, FROM skytime * skyspeed
  offset2 = the background's, FROM skytime * skyspeed2
  FOR EACH texel (x, y) OF the 128 by 128 tile
    top = the foreground layer AT ((x + offset1) BITAND 127, y)
    IF top IS transparent
      newsky[x, y] = the background layer AT ((x + offset2) BITAND 127, y)
    ELSE
      newsky[x, y] = top
  r_skymade = 1
```

**Invariants** — both offsets wrap by masking with 127, because the layers are 128 wide. Only the horizontal
axis scrolls.

The composite is 16 kilobytes and is rebuilt whenever the offsets change by a whole texel — so at the shipped
speeds, a few times per second rather than per frame.

## `R_SetSkyFrame`

**Contract** — advances the scroll clock and marks the tile stale when either offset has moved by a whole
texel.

```text
FUNCTION r_set_sky_frame()
  skytime = cl.time
  # The tile is stale when either layer's whole-texel offset has changed.
  IF either offset differs from the one the tile was built with
    r_skymade = 0
```

**Invariants** — the staleness test is the whole reason regeneration is affordable. A rebuild that composites
per frame pays sixteen kilobytes of work per frame for nothing.

## `R_GenSkyTile`, `R_GenSkyTile16`

**Contract** — copy the composited tile into a destination, for the eight- and sixteen-bit cases. Used when the
rasterizer wants the sky as an ordinary tiled surface rather than through the sky span filler.

**Notes** — the file's own comments mark four routines as needing cleanup and one as wanting a faster
unaligned copy. Nothing here is subtle beyond the layer split and the staleness test.
