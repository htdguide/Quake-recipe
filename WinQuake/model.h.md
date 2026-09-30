# WinQuake/model.h

> The in-memory shapes of all three model kinds, and the one record that holds any of them — including the rendering state the renderer writes back into the geometry.

**Needs** — [`modelgen.h`](modelgen.h.md) · [`spritegn.h`](spritegn.h.md) · [`bspfile.h`](bspfile.h.md) · [`mathlib.h`](mathlib.h.md) · [`zone.h`](zone.h.md) (the cache handle) · [`render.h`](render.h.md) (the entity-fragment type)
**Used by** — [`model.c`](model.c.md) builds all of it; every renderer file reads it; [`world.c`](world.c.md) and [`sv_main.c`](sv_main.c.md) walk the collision and visibility structures
**Tier floor** — T1 as written, because the structures are a pointer graph built in place over a loaded file and the renderer writes frame stamps into them. A T2 rebuild using index-based children meets every contract and gains bounds checking.

## Purpose

The file's own opening note is the key to it: on-disk shapes and in-memory shapes are different
types, and this file declares the second set. The loader
([`model.c`](model.c.md)) is the translation.

Two things make it more interesting than a set of records.

**The renderer writes into the geometry.** Nodes, leaves and surfaces each carry a frame stamp that
the visibility walk sets and the drawing code tests. So the "model" is not read-only data; it is
also the renderer's scratch space, and that is why a level's geometry cannot be shared between two
views (which is why the third-person camera has to re-walk rather than reuse).

**One record holds all three kinds.** A brush model, a sprite and an animated model are the same
type with a discriminant, because the client stores a flat array of models indexed by the
protocol's model number and must not care which kind arrived.

## State

The records and constants this header declares are the state; they are described in the sections below.

## Brush models — the geometry of a level

```text
RECORD MVertex
  position : vec3

RECORD MPlane                   # the in-memory plane; wider than the on-disk one
  normal   : vec3
  dist     : real
  type     : byte               # 0,1,2 axial; 3,4,5 nearest axis
  signbits : byte               # bit i set when normal[i] is negative
  pad      : byte[2]

RECORD Texture
  name             : text[16]
  width, height    : int
  anim_total       : int        # total tenths of a second in the cycle; 0 = none
  anim_min, anim_max : int      # this frame is shown for [min, max)
  anim_next        : Texture    # the cycle, as a ring
  alternate_anims  : Texture    # the SECOND cycle, for a toggled surface
  offsets          : int[4]     # to each mip level, from this record

RECORD MEdge
  v : int (16-bit unsigned)[2]
  cachededgeoffset : int        # the rasterizer's per-frame scratch

RECORD MTexInfo
  vecs      : real[2][4]        # the s and t projection
  mipadjust : real              # scales the mip selection for this surface
  texture   : Texture
  flags     : int

RECORD MSurface                 # one polygon
  visframe     : int            # the frame this was last found visible
  dlightframe  : int            # the frame a dynamic light last touched it
  dlightbits   : int            # WHICH lights, as a bit set
  plane        : MPlane
  flags        : int            # the SURF_ set below
  firstedge, numedges : int     # into the model's surface-edge array
  cachespots   : SurfCache[4]   # the surface cache, ONE PER MIP LEVEL
  texturemins  : int (16-bit)[2]   # the surface's extent in texture space,
  extents      : int (16-bit)[2]   # snapped to the 16-unit lightmap grid
  texinfo      : MTexInfo
  styles       : byte[4]        # light style per lightmap layer
  samples      : bytes          # the lightmap samples

RECORD MNode                    # an interior node; the first four fields are
  contents     : int            # SHARED with MLeaf and are always 0 here
  visframe     : int
  minmaxs      : int (16-bit)[6]
  parent       : MNode
  plane        : MPlane
  children     : MNode[2]       # either may actually be an MLeaf
  firstsurface, numsurfaces : int (16-bit unsigned)

RECORD MLeaf                    # same first four fields
  contents     : int            # always NEGATIVE here
  visframe     : int
  minmaxs      : int (16-bit)[6]
  parent       : MNode
  compressed_vis : bytes        # into the model's visibility data
  efrags       : EFrag          # entities currently touching this leaf
  firstmarksurface : list<MSurface>
  nummarksurfaces  : int
  key          : int           # a per-leaf identity used by the span sorter
  ambient_sound_level : byte[4]

RECORD Hull                     # a collision tree
  clipnodes  : list<ClipNode>   # the ON-DISK node form, used as-is
  planes     : list<MPlane>
  firstclipnode, lastclipnode : int
  clip_mins, clip_maxs : vec3   # the body size this tree was built for
```

**Invariants** — five things here a rebuild must get right.

**Nodes and leaves share a four-field prefix and are used interchangeably through a node pointer.**
The contents field discriminates: zero means a node, negative means a leaf. Every tree walk in the
renderer and in the server tests it. That is C-style structural inheritance, and a rebuild in a
language with real variants should use one, but must keep the *discriminant's* meaning because the
same test appears in the collision code over the on-disk form too.

**The plane is wider in memory than on disk**: it gains the precomputed sign bits, and its type
field is recomputed. Both exist purely to make the box-versus-plane test fast
([`mathlib.h`](mathlib.h.md#box_on_plane_side)), and the padding brings the record to a size the
assembly depends on.

**Nodes gain parent pointers.** The on-disk tree has none. They are added at load so that a leaf
can be walked upward — which is how the renderer marks the chain of nodes leading to a visible leaf
([`r_bsp.c`](r_bsp.c.md)).

**A surface carries four cache slots, one per mip level**, because the software renderer caches the
*lit, mipped* texture for a surface and a surface can be visible at more than one mip level in one
frame (across a large wall). The hardware renderer ignores them.

**A collision tree records the body size it was built for.** The map compiler shrank the geometry by
that box, and [`world.c`](world.c.md#sv_hullforentity) needs the value to correct its offset. The
sizes are *not* in the file; they are compiled into the loader, and the agreement with the map
compiler is unwritten.

**The surface texture extents are snapped to the 16-unit lightmap grid** at load, not stored. So
`texturemins` is the surface's minimum texture coordinate rounded down to a multiple of 16 and
`extents` is its span rounded up — which is what makes the lightmap's dimensions derivable.

## Surface flags

```text
CONSTANT surf_planeback      = 2     # the surface faces away from its plane's
                                     # normal
CONSTANT surf_drawsky        = 4     # the sky: drawn with the scrolling warp
CONSTANT surf_drawsprite     = 8
CONSTANT surf_drawturb       = 0x10  # a liquid: drawn with the sine warp
CONSTANT surf_drawtiled      = 0x20  # no lightmap and no 256-unit subdivision;
                                     # tile the texture directly
CONSTANT surf_drawbackground = 0x40  # fills the screen behind everything
```

**Invariants** — the tiled flag is the important one: it means "this surface has no lighting and no
surface cache", and it is set for the sky and for liquids. The rasterizer has a separate, simpler
path for those ([`d_scan.c`](d_scan.c.md)), which is why sky and water are cheap and unlit.

## Sprites

```text
RECORD MSpriteFrame
  width, height        : int
  pcachespot           : pointer      # unused
  up, down, left, right : real        # the quad's extent, derived from the
                                      # file's pixel origin
  pixels               : bytes

RECORD MSpriteGroup
  numframes : int
  intervals : list<real>
  frames    : list<MSpriteFrame>

RECORD MSpriteFrameDesc
  type     : enum { single, group }
  frameptr : MSpriteFrame OR MSpriteGroup

RECORD MSprite
  type       : int                    # the orientation rule (spritegn.h)
  maxwidth, maxheight : int
  numframes  : int
  beamlength : real
  cachespot  : pointer                # unused
  frames     : list<MSpriteFrameDesc>
```

**Invariants** — the file's pixel origin is converted at load into four **signed edge distances**,
so the renderer can build a quad without knowing the convention. That conversion is the whole
difference between the on-disk and in-memory sprite.

The frame descriptor list is what makes a sprite's frames indexable, since the file's frame table is
not a fixed-size array ([`spritegn.h`](spritegn.h.md)).

## Animated models

```text
RECORD MAliasFrameDesc
  type    : enum { single, group }
  bboxmin, bboxmax : TriVertex        # quantized
  frame   : int                       # an OFFSET, not a pointer — see below
  name    : text[16]

RECORD MAliasSkinDesc
  type       : enum { single, group }
  pcachespot : pointer
  skin       : int                    # an offset

RECORD MAliasGroupFrameDesc
  bboxmin, bboxmax : TriVertex
  frame   : int                       # an offset

RECORD MAliasGroup
  numframes : int
  intervals : int                     # an offset
  frames    : list<MAliasGroupFrameDesc>

RECORD MAliasSkinGroup
  numskins  : int
  intervals : int                     # an offset
  skindescs : list<MAliasSkinDesc>

RECORD MTriangle
  facesfront : int
  vertindex  : int[3]

RECORD AliasHeader
  model      : int                    # offsets, all of them
  stverts    : int
  skindesc   : int
  triangles  : int
  frames     : list<MAliasFrameDesc>
```

**Invariants** — **every internal reference in an animated model is an offset from the header, not
a pointer.** The file's own comment states why: animated models live in the evictable cache
([`zone.h`](zone.h.md)), so the cache may **relocate** them, and a pointer graph would be
invalidated. Offsets survive the move.

That is the single most important fact about this section, and a rebuild that stores pointers here
must either pin the data or re-fix every pointer on relocation. Offsets are simpler and the twin
recommends keeping them.

## Model effect flags

```text
CONSTANT ef_rocket  = 1     # leave a rocket trail
CONSTANT ef_grenade = 2     # leave a grenade trail
CONSTANT ef_gib     = 4     # leave a blood trail
CONSTANT ef_rotate  = 8     # spin slowly (bonus items)
CONSTANT ef_tracer  = 16    # a green split trail
CONSTANT ef_zomgib  = 32    # a small blood trail
CONSTANT ef_tracer2 = 64    # an orange split trail, and rotate
CONSTANT ef_tracer3 = 128   # a purple trail
```

**Invariants** — these are read from the **model file's own flag word**
([`modelgen.h`](modelgen.h.md)) and interpreted by the client
([`cl_main.c`](cl_main.c.md#cl_relinkentities)). So an asset tells the engine to attach an effect,
and every projectile trail in the game is configured this way rather than by game logic. A rebuild
must keep the inversion or must move every trail into the game's code — and then published content
loses its trails.

These are **distinct** from the entity effect flags in [`server.h`](server.h.md), which are set per
entity by game logic and travel over the wire. Two similarly-named bit sets with different
meanings, and confusing them is easy.

## The model record

```text
RECORD Model
  name      : text[64]
  needload  : bool            # this model's data must be (re)loaded
  type      : enum { brush, sprite, alias }
  numframes : int
  synctype  : enum { synchronized, randomized }
  flags     : int             # the effect flags above
  mins, maxs : vec3           # the volume it occupies
  radius    : real

  # --- brush model only ---
  firstmodelsurface, nummodelsurfaces : int    # this SUB-MODEL's slice of the
                                               # parent's surface array
  numsubmodels : int ;  submodels : list<DModel>
  numplanes    : int ;  planes    : list<MPlane>
  numleafs     : int ;  leafs     : list<MLeaf>     # VISIBLE leaves, excluding 0
  numvertexes  : int ;  vertexes  : list<MVertex>
  numedges     : int ;  edges     : list<MEdge>
  numnodes     : int ;  nodes     : list<MNode>
  numtexinfo   : int ;  texinfo   : list<MTexInfo>
  numsurfaces  : int ;  surfaces  : list<MSurface>
  numsurfedges : int ;  surfedges : list<int>
  numclipnodes : int ;  clipnodes : list<ClipNode>
  nummarksurfaces : int ;  marksurfaces : list<MSurface>
  hulls        : Hull[4]
  numtextures  : int ;  textures  : list<Texture>
  visdata, lightdata : bytes ;  entities : text

  # --- alias and sprite ---
  cache : CacheHandle          # reach it ONLY through Mod_Extradata
```

**Invariants** — the brush-model arrays are **shared between a map and its sub-models.** Each
sub-model is a separate record whose array pointers alias the parent map's, differing only in the
surface slice and the collision tree roots. So a door's geometry is a view into the map's, and
freeing the map frees every door.

**The `needload` flag is the reload discipline** and it differs by kind: brush models and sprites
live on the hunk and are loaded once per level, so the flag is cleared at load and only set again by
a level change; animated models live in the cache and can vanish, so the flag participates in the
reload check. See [`model.c`](model.c.md#mod_forname).

**Alias and sprite data is reached only through the accessor**, never through the handle directly,
because the cache may have evicted it. That is the contract [`zone.h`](zone.h.md) sets and this is
one of its two users.

## Entry points

**Contract** — `Mod_Init` sets up the model table. `Mod_ClearAll` marks every model as needing a
reload. `Mod_ForName` finds or loads a model by name, optionally treating failure as fatal.
`Mod_Extradata` returns an alias or sprite model's data, reloading it if the cache dropped it.
`Mod_TouchModel` marks a model recently used so it survives the next level's loading.
`Mod_PointInLeaf` locates a point in a model's rendering tree. `Mod_LeafPVS` decompresses a leaf's
visibility set.
