# WinQuake/gl_warp.c

> Sky and liquid: surfaces cut at grid boundaries at load time so a per-vertex distortion looks smooth, a two-layer scrolling sky projected from the viewer, an optional six-sided sky box, and the two image decoders the sky box needs.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`gl_model.h`](gl_model.h.md) · [`gl_warp_sin.h`](gl_warp_sin.h.md) · [Seam: Hardware 3D rasterizer](../SYSTEM-REQUIREMENTS.md#seam-hardware-3d-rasterizer) · [Seam: Palette-indexed image decoding](../SYSTEM-REQUIREMENTS.md#seam-palette-indexed-image-decoding)
**Used by** — [`gl_model.c`](gl_model.c.md) subdivides at load; [`gl_rsurf.c`](gl_rsurf.c.md) emits at draw time
**Tier floor** — none

## Purpose

Two effects that the software renderer produces inside its inner loops
([`d_scan.c`](d_scan.c.md)) and that hardware cannot: water that ripples and sky that scrolls. With no per-pixel hook,
the only place left to distort is **per vertex** — so the geometry must have enough vertices, which means cutting every
liquid and sky surface into small pieces at load time. That single consequence is what this file is about.

## State

```text
VARIABLE turbsin : real[256]              # one period of a sine, from gl_warp_sin.h
CONSTANT turbscale = 256 / 2pi            # radians to table index
VARIABLE gl_subdivide_size                # grid spacing, player-settable
VARIABLE speedscale                       # the sky's scroll offset this frame
VARIABLE solidskytexture, alphaskytexture # the two layers, split from one image
VARIABLE the six sky-box faces, and whether a box is loaded
VARIABLE skymins, skymaxs                 # the box's used extent, per face
```

## `GL_SubdivideSurface`, `SubdividePolygon`, `BoundPoly`

**Contract** — called once per liquid or sky surface at load. Recovers the surface's polygon from its edge list, then
recursively splits it along axis-aligned planes placed on a grid until every piece is smaller than the grid spacing,
and stores each piece as a vertex list with base texture coordinates.

```text
FUNCTION subdivide(verts)
  FAIL if there are too many vertices for the fixed buffers
  bounds = the polygon's extent
  FOR EACH of the three axes
    m = the midpoint on this axis, SNAPPED to the nearest grid multiple
    IF the polygon reaches less than 8 units beyond m on either side  TRY the next axis
    dist[j] = each vertex's offset from the plane at m, with dist[n] = dist[0]
    FOR EACH vertex j
      IF dist[j] >= 0  append it to the front piece
      IF dist[j] <= 0  append it to the back piece
      IF dist[j] and dist[j+1] straddle the plane
        frac = dist[j] / (dist[j] - dist[j+1])
        append the interpolated point to BOTH pieces
    subdivide(front) ;  subdivide(back) ;  RETURN
  # no axis was worth cutting: emit this piece
  store the vertices with texture coordinates = vertex . texture axes
```

**Invariants** —

- **The cutting plane is snapped to a grid, not placed at the true midpoint.** So adjacent surfaces are cut at the same
  world positions and their pieces line up; cutting at each surface's own midpoint would put a vertex on one side of a
  shared edge and not the other, and the distortion would tear the seam open. This is the load-bearing decision in the
  function.
- The **eight-unit guard** stops the recursion from producing slivers: if the cut would leave almost nothing on one
  side, the axis is skipped. Without it, a polygon whose extent straddles a grid line by a hair recurses until the
  buffers overflow.
- A vertex exactly on the plane goes into **both** pieces, which keeps both closed. That is the standard polygon-split
  rule and the `>=` / `<=` pair implements it.
- The distance array is **wrapped** — the last entry duplicates the first — so the edge from the last vertex back to
  the first is handled by the same loop body as the rest. The polygon's vertex array is likewise extended by one.
- Texture coordinates are stored **unnormalized** — as raw projections onto the texture axes, not divided by the
  texture size — because the distortion below works in texture units and divides at the end. This differs from the
  ordinary surfaces in [`gl_rsurf.c`](gl_rsurf.c.md), and mixing the conventions is a rebuild trap.
- The grid spacing is player-settable, trading vertex count for distortion smoothness. Sixty-four units is the
  documented default and matches the size the texture repeats at.
- Buffers are fixed-size and the overflow is a fatal error, not a graceful split. A rebuild should size dynamically.

## `EmitWaterPolys`

**Contract** — draws a liquid surface's pieces, offsetting each vertex's texture coordinate by a sine of the *other*
coordinate plus the current time.

```text
FUNCTION emit_water(surf)
  FOR EACH piece, FOR EACH vertex with stored coordinates (os, ot)
    s = os + turbsin[ (int)((ot * 0.125 + time) * turbscale) AND 255 ]
    t = ot + turbsin[ (int)((os * 0.125 + time) * turbscale) AND 255 ]
    emit the vertex at (s/64, t/64)
```

**Invariants** — **each coordinate is displaced by a function of the other**, which is what makes the ripple a rolling
diagonal pattern rather than a uniform slide. The same pair of expressions appears in the software renderer
([`d_scan.c`](d_scan.c.md)) and produces the same motion, so the two renderers agree on how water looks.

The sine is a **256-entry table indexed by a masked integer**, so the phase wraps for free and no trigonometry runs per
vertex. The table is a data file ([`gl_warp_sin.h`](gl_warp_sin.h.md)); a rebuild may compute it at startup, but must
keep the 256-entry quantization if it wants the identical motion.

The eighth applied to the cross coordinate sets the ripple's spatial wavelength, and the final division by 64 converts
texture units to the normalized form the library wants — the counterpart of storing them unnormalized above.

## `EmitSkyPolys`, `EmitBothSkyLayers`, `R_DrawSkyChain` (the two-layer form)

**Contract** — draws a sky surface twice: once with the opaque cloud layer and once with the transparent layer blended
over it, each with its own scroll speed, the texture coordinate for each vertex computed by projecting the direction
from the viewer to that vertex onto a flattened dome.

```text
FUNCTION emit_sky(surf, scroll)
  FOR EACH piece, FOR EACH vertex
    dir = vertex - view origin
    dir.z *= 3                       # flatten the dome
    length = 6 * 63 / |dir|          # a fixed radius over the distance
    s = (scroll + dir.x * length) / 128
    t = (scroll + dir.y * length) / 128
    emit the vertex at (s, t)
```

**Invariants** —

- **The sky's texture coordinate depends on the direction from the viewer, not on the surface's position**, so the sky
  appears infinitely far away and does not slide as the player walks. That is the whole illusion, and it is why sky
  surfaces need their own path at all.
- The vertical component is **tripled before normalizing**, which flattens the dome so the clouds stretch toward the
  horizon instead of converging at a point overhead. The factor is a look, chosen by eye.
- **Two layers scroll at different speeds**, derived from the same clock with different multipliers, and the upper one
  is drawn with its background masked out. Two speeds over two layers is what makes a flat texture read as depth, and
  it costs one extra pass.
- The layers are **split from a single map texture at load** (see below), so the map format stores one image and the
  renderer makes two.
- Subdivision matters here for the same reason as water: the projection is nonlinear, so a large quad would map the sky
  as a flat plane.

## `R_InitSky`

**Contract** — takes the map's sky texture, whose left half is the opaque layer and whose right half is the transparent
one; uploads each half as its own texture, marking the transparent layer's background colour as fully transparent.

**Invariants** — the **half-and-half convention is part of the map format**, not of this code: the compiler writes one
double-width image. The transparency key is a specific palette index, and every pixel of that index becomes
transparent. A rebuild must use the same index or the clouds acquire a solid background.

## `R_LoadSkys`, `DrawSkyPolygon`, `ClipSkyPolygon`, `R_ClearSkyBox`, `MakeSkyVec`, `R_DrawSkyBox`, `R_DrawSkyChain` (the box form)

**Contract** — the alternative sky: load six images named by face, then, instead of drawing the sky surfaces, *project*
each of them onto the six faces of a cube around the viewer, accumulate how much of each face was covered, and finally
draw only the covered part of each face as a single quad at a great distance.

```text
FUNCTION clip_sky_polygon(poly, remaining_faces)
  # Clip the polygon against the six planes that separate cube faces,
  # recursing on each side, until it lies within one face.
  IF no faces remain  DISCARD
  FOR EACH remaining face plane
    classify every vertex as front, back or on
    IF all on one side  continue to the next plane
    split the polygon and recurse on both halves with that plane removed
    RETURN
  draw_sky_polygon(poly)             # lies in one face: record its extent

FUNCTION draw_sky_polygon(poly)
  face = the axis of largest absolute coordinate, and its sign
  FOR EACH vertex
    project onto that face's two in-plane axes, divided by the dominant one
    grow skymins/skymaxs for that face to include the projected point
```

**Invariants** —

- The sky surfaces are used only to **decide which parts of which cube faces are visible**; nothing of their geometry is
  drawn. So a map with a sky visible through a small window draws six tiny quads instead of six full ones. That is the
  whole reason for the clipping: correctness would be served by always drawing all six faces, and this is an
  optimization that pays when the sky is barely visible.
- The extents are reset each frame and accumulated across every sky surface, so the drawing happens **once after the
  whole chain**, not per surface.
- The cube's corners are pushed out to a large fixed distance and the coordinates are pulled slightly inward from each
  face's edge, because otherwise the seams between faces show the texture's clamped edge. The inward nudge is the
  standard sky-box seam fix.
- The box takes priority over the two-layer sky when its images loaded, and is selected by naming a set of files. The
  original ships no such files, so this is an unused feature — recorded because it is a complete sky-box implementation
  and because the clipping idea generalizes.

## `LoadPCX`, `LoadTGA`, `fgetLittleShort`, `fgetLittleLong`

**Contract** — decode a run-length-encoded palette image, and decode an uncompressed or run-length-encoded true-colour
image, each into a buffer with its dimensions. Reject anything outside the small set of variants actually used.

**Invariants** — these exist **only** for the sky box, which is why they live in this file rather than a shared image
module. Both formats are trivially rebuildable and the recipe treats them as such
([Seam: Palette-indexed image decoding](../SYSTEM-REQUIREMENTS.md#seam-palette-indexed-image-decoding)); a rebuild will more likely use its
platform's image loader and delete both.

The multi-byte reads are explicitly little-endian, because these are file formats and the engine runs on machines of
either byte order — the same rule as [`common.c`](common.c.md).

**Notes** — the two sky mechanisms and the water share only this file, not any code. What they share is the *problem*:
hardware gives no per-pixel hook, so anything the software renderer did inside a span loop must be re-expressed as
geometry, and geometry must be subdivided to express anything curved. A rebuild with programmable shading does all of
this per pixel and throws away the subdivision, the vertex counts and the fixed-size buffers — but should keep the
grid-snapped cut idea in mind, because it is the general answer whenever adjacent surfaces must be split consistently.
