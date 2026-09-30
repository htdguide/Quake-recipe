# WinQuake/modelgen.h

> The animated-model format: a shared vertex set quantized to one byte per axis, per-frame positions, and a per-vertex index into a fixed normal table.

**Needs** — [`mathlib.h`](mathlib.h.md)
**Used by** — [`model.c`](model.c.md) and [`gl_model.c`](gl_model.c.md) (the loader) · [`r_alias.c`](r_alias.c.md) and [`gl_mesh.c`](gl_mesh.c.md) (the renderers) · [`anorms.h`](anorms.h.md) supplies the normal table this format indexes
**Tier floor** — none; a byte layout

## Purpose

Every animated thing in the game — every monster, every weapon in hand, every gibbed
limb — is one of these. The format's whole character comes from one decision: a
vertex position is **three bytes**, not three floats, interpreted through a per-model
scale and origin. That makes a 200-frame monster fit in a few hundred kilobytes and
makes frame interpolation a byte-lerp, and it is why models in this engine visibly
shimmer when close to the camera.

The second decision is that lighting is per vertex and *precomputed as a direction
index*: a vertex carries a one-byte index into a fixed table of 162 unit normals,
rather than a normal of its own. The renderer then looks up a brightness for that
index and the current light direction from a precomputed table
([`anorm_dots.h`](anorm_dots.h.md)). So model lighting costs one table lookup per
vertex, with no arithmetic at all.

## State

A file format. No runtime state.

## The file

```text
RECORD ModelHeader
  ident          : int          # the four bytes "IDPO", little-endian
  version        : int          # must be 6
  scale          : vec3         # multiply a quantized vertex by this
  scale_origin   : vec3         # then add this
  boundingradius : real
  eyeposition    : vec3         # where a viewing entity's eyes sit
  numskins       : int
  skinwidth      : int          # every skin is this size
  skinheight     : int
  numverts       : int          # the SHARED vertex count; every frame has
                                # exactly this many
  numtris        : int
  numframes      : int
  synctype       : int          # 0: animation phase is synchronized to the
                                # global clock; 1: randomized per entity
  flags          : int          # effects the engine applies: a rocket trail,
                                # a gib trail, rotation, and so on
  size           : real         # average triangle area; unused by the engine
```

Then, in order:

```text
# 1. Skins: `numskins` entries, each either single or a group
RECORD SkinType      { type : int }            # 0 single, 1 group
#   single: skinwidth * skinheight bytes of palette indices
#   group:  a count, then that many intervals (one real each),
#           then that many bitmaps

# 2. Texture coordinates: `numverts` entries, one per SHARED vertex
RECORD STVert
  onseam : int                  # non-zero: this vertex lies on the model's
                                # front/back seam
  s, t   : int                  # in skin pixels

# 3. Triangles: `numtris` entries
RECORD Triangle
  facesfront : int              # non-zero: use the front-facing texture
                                # coordinates for this triangle's vertices
  vertindex  : int[3]           # indices into the shared vertex set

# 4. Frames: `numframes` entries, each either single or a group
RECORD FrameType     { type : int }            # 0 single, 1 group
#   single: a Frame header, then `numverts` TriVertex records
#   group:  a count, a group bounding box, then that many intervals,
#           then that many (Frame header + `numverts` TriVertex) blocks

RECORD Frame
  bboxmin, bboxmax : TriVertex  # quantized; the normal index is unused here
  name             : text[16]

RECORD TriVertex
  v                : byte[3]    # quantized position
  lightnormalindex : byte       # 0..161, into the fixed normal table
```

## Invariants a rebuild must honour

**Dequantization.** A vertex's world position within the model is
`v * scale + scale_origin`, component-wise. So the model occupies a box of
`255 * scale` units and the quantization step is `scale` — typically a quarter to a
half unit on a player-sized model, which is exactly the visible shimmer.

**One vertex set, many frames.** Texture coordinates and triangle indices are given
once, and each frame supplies only positions. So a frame cannot change topology, and
interpolation between any two frames of the same model is well defined without
correspondence work. This is the reason frame blending is cheap and the reason models
cannot change their silhouette structure.

**The seam.** A model's skin is a front view and a back view side by side. A vertex on
the boundary belongs to both, so it carries one texture coordinate and a flag; a
triangle that faces away from the viewer adds half the skin width to the coordinate of
every seam vertex it touches. The flag and the per-triangle front-facing bit exist
solely to make that adjustment decidable without duplicating vertices.

```text
FUNCTION skin_coord_for(triangle, vertex) -> (s, t)
  s = stvert[vertex].s ;  t = stvert[vertex].t
  IF NOT triangle.facesfront AND stvert[vertex].onseam
    s = s + skinwidth / 2         # use the back half of the skin
  RETURN (s, t)
```

**Lighting is an index, not a normal.** The normal table has 162 entries and is
compiled into the engine ([`anorms.h`](anorms.h.md)); the file only indexes it. A
rebuild must use the *same* table, because the indices in published models refer to it
by position. The table itself is an icosahedral subdivision of the sphere; its values
are given in the twin.

**Frame groups have their own timing.** A group carries one interval per frame, giving
the time at which that frame *ends*, cumulative from the group's start. So playback is
"find the first interval greater than the elapsed time". The group also carries its own
bounding box, which is the union of its frames'.

**Synchronization type.** With the synchronized setting every instance of the model
animates from the same global clock, so a row of torches flickers in unison; with the
randomized setting each entity gets a random phase offset at spawn. A rebuild that
omits this makes every flame in a level identical, which is immediately visible.

**Flags carry engine behaviour, not model data.** The flag word asks the engine to
attach a rocket trail, a gib trail, a tracer, or constant rotation to any entity using
this model. That is a genuine layering inversion — the asset tells the engine what
effect to run — and it is how every projectile trail in the game is configured. See
[`cl_main.c`](cl_main.c.md#cl_relinkentities).

**Skin dimensions are per model, not per skin.** Every skin in one file is the same
size, and it need not be a power of two, which is why the hardware renderer resamples
skins at load.

**Multiple skins are a palette, not an animation** — usually. A model with several
single skins is choosing between them by the entity's skin number; a model with a skin
*group* animates that slot on the group's own intervals.

## Notes

The header's average-triangle-area field is written by the model compiler and never
read by the engine.

The format has no bounds check anywhere: the loader must validate `numverts`,
`numtris` and `numframes` against the file's actual length itself, and the original
does not. A malicious model crashes the original engine; a rebuild should validate.

Nothing in this format is compressed, and the whole file is loaded and kept. A
200-frame monster at 300 vertices is 240 kilobytes of vertex data, which is why models
live in the evictable cache rather than on the hunk
([`model.c`](model.c.md#mod_loadaliasmodel)).
