# WinQuake/bspfile.h

> The compiled map format: fifteen lumps holding a rendering tree, three collision trees at different body sizes, precomputed visibility, and baked lighting.

**Needs** — [`mathlib.h`](mathlib.h.md) (for the vector type in the utility half)
**Used by** — [`model.c`](model.c.md) and [`gl_model.c`](gl_model.c.md) (the loaders) · [`world.c`](world.c.md) (collision) · [`r_bsp.c`](r_bsp.c.md) and [`gl_rsurf.c`](gl_rsurf.c.md) (traversal) · [`sv_main.c`](sv_main.c.md) (visibility) · [`quakedef.h`](quakedef.h.md) includes it for everyone
**Tier floor** — none; a byte layout

## Purpose

A map is not geometry with a collision mesh bolted on. It is a *binary space
partition* computed offline, and almost everything the engine does at run time — decide
what to draw, decide what a bullet hit, decide what a player can see, decide what
lights a wall — is a walk of one of the trees in this file. The offline precomputation
is the engine's central performance decision, and this format is its output.

Four things are precomputed here that a naive engine would compute at run time, and
each is worth a rebuild's attention:

- **Which surfaces are potentially visible from where.** A per-leaf bitset over all
  leaves, run-length encoded. It turns "what might I see" from a geometric query into
  a byte-array lookup.
- **Collision at three body sizes.** Rather than inflate obstacles by the player's
  radius at run time, the compiler builds three separate trees, each already shrunk by
  one of three fixed bounding boxes. A trace is then a point trace.
- **Static lighting, per surface, per light style.** Sampled on a 16-unit grid and
  stored as one byte per sample.
- **Plane classification.** Each plane records whether it is axis aligned and the sign
  pattern of its normal, so the hot side test can branch instead of compute.

## State

This is a file format, not a module. No runtime state.

## The file

```text
RECORD Header
  version : int (32-bit)        # must be 29
  lumps   : Lump[15]

RECORD Lump
  fileofs : int                 # byte offset from the start of the file
  filelen : int                 # length in bytes
```

Every integer and float in the file is little-endian. The lumps may appear in any
order in the file; the directory is authoritative. The fifteen lumps, in directory
order:

| # | Lump | Holds |
|---|---|---|
| 0 | Entities | one text block: the entity list, in the brace-and-quoted-pair syntax the shared tokenizer reads |
| 1 | Planes | the plane array both trees index |
| 2 | Textures | the embedded texture archive |
| 3 | Vertexes | positions |
| 4 | Visibility | run-length encoded per-leaf visibility bitsets |
| 5 | Nodes | the rendering tree's interior nodes |
| 6 | Texinfo | texture mapping vectors and flags |
| 7 | Faces | polygons |
| 8 | Lighting | lightmap samples, one byte each |
| 9 | Clipnodes | the collision trees' interior nodes, all three interleaved by root |
| 10 | Leafs | the rendering tree's leaves |
| 11 | Marksurfaces | indices from a leaf into the face array |
| 12 | Edges | vertex pairs |
| 13 | Surfedges | signed indices into the edge array |
| 14 | Models | sub-models: the world plus each moving brush |

## Records

```text
RECORD Model                    # one sub-model: index 0 is the world
  mins, maxs : vec3             # bounding box
  origin     : vec3
  headnode   : int[4]           # root node index per hull; see below
  visleafs   : int              # leaves in this sub-model, excluding leaf 0
  firstface, numfaces : int

RECORD Plane
  normal : vec3
  dist   : real                 # the plane is { p : dot(normal,p) = dist }
  type   : int                  # 0,1,2 = axial +x,+y,+z; 3,4,5 = nearest axis

RECORD Node                     # rendering tree, interior
  planenum    : int
  children    : int (16-bit signed)[2]   # >=0 is a node index;
                                         # negative is -(leaf index + 1)
  mins, maxs  : int (16-bit signed)[3]   # integer bounding box, for sphere cull
  firstface   : int (16-bit unsigned)
  numfaces    : int (16-bit unsigned)    # faces on BOTH sides of the plane

RECORD ClipNode                 # collision tree, interior
  planenum : int
  children : int (16-bit signed)[2]      # >=0 is a clipnode index;
                                         # negative is a CONTENTS value

RECORD Leaf                     # rendering tree, leaf
  contents          : int                # a CONTENTS value
  visofs            : int                # offset into the visibility lump,
                                         # or -1 for none
  mins, maxs        : int (16-bit signed)[3]
  firstmarksurface  : int (16-bit unsigned)
  nummarksurfaces   : int (16-bit unsigned)
  ambient_level     : byte[4]            # water, sky, slime, lava loudness

RECORD Face
  planenum : int (16-bit)
  side     : int (16-bit)       # non-zero: the face faces away from the normal
  firstedge : int               # into surfedges; 32-bit because there are
                                # more than 65535 of them
  numedges : int (16-bit)
  texinfo  : int (16-bit)
  styles   : byte[4]            # light style per lightmap layer; 255 = unused
  lightofs : int                # byte offset into the lighting lump, or -1

RECORD TexInfo
  vecs    : real[2][4]          # s and t: a 3-vector and an offset
  miptex  : int                 # index into the embedded texture archive
  flags   : int                 # bit 0: special — sky or liquid; no lightmap
                                # and no 256-unit subdivision

RECORD Edge
  v : int (16-bit unsigned)[2]  # vertex indices

RECORD Vertex
  point : real[3]

RECORD MipTexLump               # the texture lump's own header
  nummiptex : int
  dataofs   : int[nummiptex]    # offsets to each Texture, from the lump start;
                                # -1 means the texture is not embedded and must
                                # be found in an external texture archive

RECORD Texture
  name    : text[16]            # null-padded; a leading '*' means liquid,
                                # "sky" prefix means sky, '+' means animated
  width, height : int (unsigned)# both multiples of 16
  offsets : int[4]              # to each mip level, from this record's start
```

## Invariants that a rebuild must honour

**Version 29.** Every published map is this version. There is no forward or backward
compatibility path.

**Face winding.** A face's edges are given as signed indices into the surface-edge
array; a positive index means the edge's vertices in order, a negative index means
`-index` traversed backwards. **Edge 0 is never referenced**, precisely because zero
has no sign. The resulting vertex loop is convex and planar.

**Node children encoding.** A non-negative child is a node index; a negative child *n*
is leaf number `-(n+1)`. So child −1 is leaf 0, which is the solid leaf. Because the
field is a signed 16-bit value, there can be at most 32767 nodes and at most 32767
leaves.

**Leaf 0 is solid and has no visibility.** Every solid region of the map resolves to
it, so a trace that lands in leaf 0 is inside a wall. It is the only leaf with no
visibility bitset, and the per-sub-model leaf count excludes it.

**Clipnode children encoding differs.** A negative clipnode child is not a leaf index
— it is a contents value directly. So a collision trace terminates with a contents
value in hand rather than with a leaf to look up.

**Three collision hulls, plus the rendering tree.** Each sub-model's four head-node
indices are: index 0, the rendering tree's root; indices 1, 2 and 3, the roots of
three clipnode trees built for three fixed bounding boxes. The map compiler shrank the
geometry by each box, so a trace for an entity of that size is a **point** trace. The
box sizes are not in this file — they are compiled into
[`world.c`](world.c.md#sv_hullforentity) and must agree with whatever the map compiler
used. That agreement is an unwritten contract between two programs and a rebuild must
adopt the same numbers.

**Contents values are negative and ordered.** Empty is −1, solid −2, water −3, slime
−4, lava −5, sky −6, and six values naming a current direction follow. Two more —
origin and clip — exist only inside the compiler and never appear in a finished map.
The *ordering* is load-bearing: the trace's "which content wins where two meet" logic
and the liquid-damage logic both compare them numerically.

**Plane type and its cousin.** Types 0 through 2 mean the normal is exactly along +x,
+y or +z; 3 through 5 mean it is merely closest to that axis. Only the first three
enable the fast side test ([`mathlib.h`](mathlib.h.md#box_on_plane_side)). The source's
own comment observes that the field is trivial to regenerate at load time, and a
rebuild should do that rather than trust the file.

**Lightmap layout.** A face has up to four light *styles*; its lighting offset points
at that many consecutive sample blocks. A sample grid is one sample per 16 world units
along each texture axis, so the block's dimensions follow from the face's extent and
are computed at load, not stored. An unused style slot holds 255. A face with offset −1
is unlit — fullbright.

**Texture names carry semantics in their first character.** A leading asterisk means a
liquid surface; the prefix `sky` means the sky; a leading plus introduces an animation
group whose second character is a frame number and where a `+a` variant is an
alternate sequence. This is string-based dispatch in a binary format, and every engine
that reads these maps implements it. See
[`model.c`](model.c.md#mod_loadtextures).

**Visibility is run-length encoded over zero bytes only.** A zero byte in the stream is
followed by a count of zero bytes to emit; any other byte is literal. Decoding needs
the leaf count to know when to stop. See [`model.c`](model.c.md#mod_decompressvis).

**Texture dimensions are multiples of 16** so that four mip levels exist and each
halves cleanly. The software renderer's surface cache depends on it.

## Design bounds

The file declares the map compiler's capacities: 256 sub-models, 32767 planes, 32767
nodes, 32767 clipnodes, 8192 leaves, 65535 vertices, 65535 faces, 65535 mark surfaces,
4096 texture mappings, 256000 edges, 512000 surface edges, 512 textures, 2 MB of
texture data, 1 MB of lighting, 1 MB of visibility, 65536 bytes of entity text, 1024
entities.

**Notes** — these are the *compiler's* limits, not the engine's; the engine allocates
from the file's actual sizes. They are recorded here because a rebuild writing its own
map compiler must respect the ones the format enforces — the 16-bit index fields make
the node, clipnode, leaf, vertex, face and mark-surface limits hard, and the rest are
soft.

## The utility half

The latter half of the file is compiled only into the map tools, not into the engine:
fixed-size global arrays for every lump, whole-file load and save, the entity-text
parser and printer, and the visibility run-length codec. A rebuild of the *engine*
ignores it; a rebuild of the *toolchain* needs it, and the codec's decode side is also
in [`model.c`](model.c.md).

**Notes** — the tools using fixed global arrays sized to the design bounds, while the
engine allocates from actual sizes, is the one structural difference between the two
consumers of this format. It explains why the bounds are here at all.
