# WinQuake/model.c

> Loads all three model kinds, turning on-disk shapes into the renderer's pointer graph; builds the animation cycles from texture names; derives the three collision hulls; and manages the one model table the client and server share.

**Needs** — [`model.h`](model.h.md) · [`bspfile.h`](bspfile.h.md) · [`modelgen.h`](modelgen.h.md) · [`spritegn.h`](spritegn.h.md) · [`quakedef.h`](quakedef.h.md) · [`r_local.h`](r_local.h.md) (the placeholder texture, the destination pixel width, the sky setup) · [`zone.h`](zone.h.md) · [`common.h`](common.h.md) · [`mathlib.h`](mathlib.h.md) · [`cmd.h`](cmd.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`sv_main.c`](sv_main.c.md) and [`pr_cmds.c`](pr_cmds.c.md) (the server's precache) · [`cl_parse.c`](cl_parse.c.md) (the client's) · [`world.c`](world.c.md) (collision hulls) · every renderer file · [`host_cmd.c`](host_cmd.c.md)
**Tier floor** — T1 as written; a T2 rebuild using index-based references throughout meets every contract and gains validation

## Purpose

The file's own opening line is the structural fact worth knowing: **models are the only resource
shared between a client and a server in the same process.** One table, one loader, and both halves
of the engine index into it. That is why the model record carries both collision hulls (server) and
surface caches (renderer), and why a listen server loads each model once.

Four parts deserve attention beyond the mechanical byte-swapping.

**The texture animation sequencer** builds cycles out of *names*: a texture called `+0lava` is frame
zero of a cycle, `+a` through `+j` is an alternate cycle for the same base name, and the loader
threads them into two rings. It is string-based dispatch recovering structure the file format does
not record.

**The collision hulls' body sizes are here, not in the map.** Three hard-coded boxes must agree with
whatever the map compiler used, and that agreement is unwritten anywhere.

**A sub-model is a shallow copy of its parent** with two fields changed, which is why a door's
geometry aliases the map's.

**An animated model is relocated into the evictable cache** after being built on the hunk, which is
why every internal reference in it is an offset rather than a pointer.

## State

```text
CONSTANT max_mod_known = 256
VARIABLE mod_known   : Model[256]
VARIABLE mod_numknown : int
VARIABLE loadmodel   : Model        # the model being loaded; a global, so the
                                    # loaders take fewer arguments
VARIABLE loadname    : text[32]     # its base name, used as the hunk tag
VARIABLE mod_base    : bytes        # the file being parsed
VARIABLE mod_novis   : bytes        # an all-ones visibility set

# Values for the reload state
CONSTANT nl_present       = 0
CONSTANT nl_needs_loaded  = 1
CONSTANT nl_unreferenced  = 2
```

**Invariants** — the reload state is **three-valued, not a boolean**, and the distinction carries
the level-change discipline: *present* means loaded and in use, *needs loaded* means named but not
yet read, and *unreferenced* means loaded but not touched since the last level change — hence
reusable. [`Mod_ClearAll`](#mod_clearall) sets everything to unreferenced and the precache pass
then either reloads or revives each model.

The loader communicates through globals rather than arguments, which makes it non-reentrant. It
never needs to be.

## Table management

### `Mod_Init`

**Contract** — fills the all-visible set with ones.

**Invariants** — that set is returned for a leaf with no visibility information, so a map compiled
without visibility data renders everything. It is also returned for **leaf 0**, the solid leaf,
which is what makes a viewpoint inside a wall see the whole level rather than nothing.

### `Mod_FindName`

**Contract** — takes a name; returns the existing model record with that name, or claims a free or
reusable slot for it and marks it as needing loading. An empty name is a fatal error. When the table
is full, reuses an unreferenced slot, preferring a non-animated one, and frees its cache entry
first. A full table with nothing unreferenced is a fatal error.

```text
FUNCTION mod_find_name(name) -> Model
  IF name is empty  FAIL WITH "NULL name"
  available = nothing
  FOR EACH known model mod
    IF mod.name == name  RETURN mod                    # exact match
    IF mod.needload IS unreferenced
      IF available IS nothing OR mod.type IS NOT alias
        available = mod                                # prefer a NON-alias slot
  IF the table is full
    IF available EXISTS
      mod = available
      IF mod.type IS alias AND its cache entry is live  release it
    ELSE FAIL WITH "mod_numknown == MAX_MOD_KNOWN"
  ELSE
    claim the next slot
  mod.name = name ;  mod.needload = needs_loaded
  RETURN mod
```

**Invariants** — the preference for recycling a **non-animated** slot is deliberate: animated models
live in the cache and their data may still be resident, so recycling one throws away work that a
later level might reuse. Recycling a brush model or sprite costs nothing because their data is on
the hunk and has already been freed by the level change.

The scan is linear over up to 256 entries and runs once per precache name per level.

### `Mod_TouchModel`

**Contract** — takes a name; finds its record and, if it is a resident animated model, marks its
cache entry as recently used.

**Invariants** — this is what keeps the models a level will need from being evicted *while that
level loads*. The precache pass touches every name before loading any data, so the cache's
least-recently-used order favours the incoming level over the outgoing one
([`zone.h`](zone.h.md)).

### `Mod_ClearAll`

**Contract** — marks every known model as unreferenced. Additionally clears the cache handle of
every sprite.

**Invariants** — clearing the sprite handles is marked in the source as a fix for cache allocation
errors, and the reason is that a sprite's data is on the *hunk* even though it is reached through a
cache handle ([`Mod_LoadSpriteModel`](#mod_loadspritemodel) assigns the handle directly). The level
change frees the hunk, so the handle becomes a pointer into freed memory; without this clearing, the
next cache operation would follow it. A rebuild should give sprites their own storage discipline and
the anomaly disappears.

### `Mod_LoadModel`

**Contract** — takes a record and whether failure is fatal; ensures its data is resident. Returns
early if it already is — for an animated model by checking the cache, for the others by checking the
reload state. Otherwise reads the file, derives the hunk tag from its base name, and dispatches on
the first four bytes to one of three loaders. A missing file is either fatal or returns nothing.

```text
FUNCTION mod_load_model(mod, crash) -> optional<Model>
  IF mod.type IS alias
    IF its cache entry is live  mod.needload = present ;  RETURN mod
  ELSE
    IF mod.needload IS present  RETURN mod

  buf = load mod.name, into a 1024-byte stack buffer if it fits
  IF not found
    IF crash  FAIL WITH "<name> not found"
    RETURN nothing
  loadname = the base name OF mod.name        # the 8-character hunk tag
  loadmodel = mod
  mod.needload = present
  SELECT the first four bytes, little-endian
    "IDPO"    mod_load_alias_model(mod, buf)
    "IDSP"    mod_load_sprite_model(mod, buf)
    otherwise mod_load_brush_model(mod, buf)
  RETURN mod
```

**Invariants** — the **format is decided by a magic number, and the default is a map.** A map has no
magic; its first field is a version number. So an unrecognized file is treated as a map and fails on
the version check — which produces a misleading error for a corrupt model.

The two residency tests differ because the two storage disciplines differ. An animated model's
reload state can say *present* while its cache entry has been evicted, so the cache is authoritative
for it.

A small file is read into the caller's stack buffer, avoiding a temporary hunk allocation and —
as the source notes — avoiding dirtying the cache region. The 1024-byte threshold covers only the
smallest models.

### `Mod_ForName`

**Contract** — find or create the record, then load it.

### `Mod_Extradata`

**Contract** — takes a record; returns its cached data, reloading it if evicted. A reload that still
produces nothing is a fatal error.

**Invariants** — the **only** legitimate way to reach an animated model's data. Every renderer call
site goes through it, and the reload is transparent — which is what makes cache eviction invisible
to the renderer.

### `Mod_Print`

**Contract** — the `mcache` console command; lists every known model, its cached address, and
whether it is unreferenced or awaiting load.

## Visibility

### `Mod_PointInLeaf`

**Contract** — takes a point and a model; descends the rendering tree to the leaf containing the
point. A model with no tree is a fatal error.

```text
FUNCTION mod_point_in_leaf(p, model) -> MLeaf
  node = the model's root
  WHILE node is not a leaf                  # contents >= 0 means a node
    d = dot(p, node.plane.normal) - node.plane.dist
    node = node.children[0] IF d > 0 ELSE node.children[1]
  RETURN node AS a leaf
```

**Invariants** — a point exactly on a plane goes to the **back** child here (`d > 0` selects front),
which is the opposite of the collision tree's rule
([`world.c`](world.c.md#sv_hullpointcontents), where `d < 0` selects back so equality goes front).
Two tree walks with opposite tie-breaking, in one engine. Neither is wrong; a rebuild should keep
both as written, because the visibility and collision answers are consumed independently.

### `Mod_DecompressVis`

**Contract** — takes a compressed visibility run and a model; expands it into a shared static buffer
and returns it. A null input produces an all-visible set.

```text
FUNCTION mod_decompress_vis(in, model) -> bytes
  row = (model's leaf count + 7) SHIFTED RIGHT 3       # bytes needed
  IF in IS nothing
    fill `row` bytes with ones ;  RETURN the buffer
  out = the buffer
  REPEAT
    IF the next input byte IS NOT zero
      copy it ;  CONTINUE                     # a literal byte
    # A zero byte introduces a run: the next byte is the count of zeros.
    c = the byte after it ;  advance past both
    emit c zero bytes
  WHILE fewer than `row` bytes have been emitted
  RETURN the buffer
```

**Invariants** — the encoding compresses **runs of zero bytes only**. A zero byte is always followed
by a count; any other byte is literal. That is the whole codec and it is symmetric with the map
compiler's.

The loop is bounded by the *output* count, so a malformed stream reads past the input's end. And
the buffer is shared and static, so **one decompressed set is live at a time** — which is why
[`sv_main.c`](sv_main.c.md#sv_addtofatpvs) copies the result before recursing.

The row count rounds the bit count up to a byte, so trailing bits beyond the leaf count are
whatever the encoding produced.

### `Mod_LeafPVS`

**Contract** — takes a leaf and a model; returns its visibility set, or the all-visible set when the
leaf is leaf 0.

**Invariants** — leaf 0 is the solid leaf and has no set, so a viewpoint inside geometry sees
everything. That is a deliberate fallback, not a bug: it keeps the renderer working when a player is
briefly inside a wall.

## Loading a map

### `Mod_LoadBrushModel`

**Contract** — reads a map. Validates the version, byte-swaps the header, loads fifteen lumps in a
specific order, derives the rendering tree's collision hull, then walks the sub-model table
creating a separate model record for each — the first overwriting the record being loaded and the
rest claimed by name.

```text
FUNCTION mod_load_brush_model(mod, buffer)
  mod.type = brush
  IF the version is not 29  FAIL WITH "wrong version number"
  mod_base = buffer
  byte-swap every 32-bit word of the header

  # The load ORDER is a dependency order, not the lump order:
  vertexes ;  edges ;  surfedges          # geometry the faces index
  textures                                # which faces need
  lighting                                # which faces point into
  planes                                  # which faces and nodes reference
  texinfo                                 # which needs textures
  faces                                   # which needs all of the above
  marksurfaces                            # which needs faces
  visibility                              # which leaves point into
  leafs                                   # which needs marksurfaces and
                                          # visibility
  nodes                                   # which needs leafs and planes
  clipnodes
  entities
  submodels

  mod_make_hull0()                        # derive hull 0 from the draw tree
  mod.numframes = 2                       # regular and alternate texture
                                          # animation
  mod.flags = 0

  FOR EACH sub-model i
    bm = submodels[i]
    mod.hulls[0].firstclipnode = bm.headnode[0]
    FOR EACH hull j FROM 1 TO 3
      mod.hulls[j].firstclipnode = bm.headnode[j]
      mod.hulls[j].lastclipnode  = mod.numclipnodes - 1
    mod.firstmodelsurface = bm.firstface
    mod.nummodelsurfaces  = bm.numfaces
    mod.mins, mod.maxs = bm's bounds ;  mod.radius = from those bounds
    mod.numleafs = bm.visleafs
    IF i is not the last
      # Claim the record named "*<i+1>" and SHALLOW-COPY this one into it.
      loadmodel = mod_find_name("*" + (i+1))
      copy the WHOLE record ;  restore its own name
      mod = loadmodel                     # continue filling THAT one
```

**Invariants** — three decisions here shape everything downstream.

**The load order is a dependency order.** Faces resolve pointers into textures, lighting, planes,
texinfo, vertices and edges, so all six must exist first. Leaves resolve into marksurfaces and
visibility. Nodes resolve into leaves. A rebuild that loads in lump order gets null pointers.

**A sub-model record is a shallow copy of the map's**, with only the surface slice, the four hull
roots, the bounds and the leaf count changed. So every array pointer aliases the parent's, and
freeing the map frees every door. That is why the whole level's geometry is one hunk allocation
group and why a door cannot outlive its map.

**The record being loaded becomes sub-model 0** — the map itself — and the loop's copy step happens
*before* moving on, so the sequence is: fill the map's record, copy it to `*1`, fill `*1`'s
specifics, copy to `*2`, and so on. The source's own comment calls this confusing, and it is: each
iteration fills the *previous* iteration's record.

**Hull 0 is derived, not loaded.** The map file has three collision trees; the fourth "hull" is the
rendering tree converted into collision-node form
([`Mod_MakeHull0`](#mod_makehull0)), so that a point trace can use it. That is how
[`SV_PointContents`](world.c.md#sv_pointcontents-sv_truepointcontents) works at all.

The frame count of 2 is the texture animation's two cycles, not geometry frames.

### `Mod_LoadVertexes`, `Mod_LoadEdges`, `Mod_LoadSurfedges`, `Mod_LoadMarksurfaces`, `Mod_LoadLighting`, `Mod_LoadVisibility`, `Mod_LoadEntities`

**Contract** — each reads one lump into hunk memory, byte-swapping as it goes. A lump length that is
not a multiple of its record size is a fatal error. An empty lump leaves the corresponding pointer
absent. The mark-surface loader validates each index against the face count; the others validate
nothing.

**Invariants** — the edge loader allocates **one more record than the file holds**. Nothing writes
it. It is headroom for the negative-index convention ([`bspfile.h`](bspfile.h.md)), where edge 0 is
never referenced — a belt-and-braces allocation, not a requirement.

The mark-surface loader is the only one with a bounds check, and it is the only one whose indices
come from a lump rather than being derived. A rebuild should check all of them.

### `Mod_LoadPlanes`

**Contract** — reads the plane array, byte-swapping and computing each plane's sign bits.

```text
FUNCTION mod_load_planes(lump)
  # NOTE: allocates TWICE the needed space.
  allocate count * 2 records
  FOR EACH plane
    bits = 0
    FOR EACH axis j
      normal[j] = little-endian float
      IF normal[j] < 0  set bit j OF bits
    dist = little-endian float
    type = little-endian int                # taken from the file, NOT recomputed
    signbits = bits
```

**Invariants** — the sign bits are computed here so that the box-versus-plane test can branch on
them ([`mathlib.c`](mathlib.c.md#boxonplaneside)). The plane's axis classification is **taken from
the file** rather than recomputed, despite [`bspfile.h`](bspfile.h.md)'s own note that it is trivial
to regenerate. A rebuild should recompute it and stop trusting the file.

The double allocation is unexplained and unused. Harmless waste; a rebuild allocates once.

### `Mod_LoadTextures`

**Contract** — reads the embedded texture archive: validates that each texture's dimensions are
multiples of 16, copies its header and all four mip levels into one hunk block with the offsets
rebased, and initializes the sky's two-layer texture when the name begins with the sky prefix. Then
builds the animation cycles.

```text
FUNCTION mod_load_textures(lump)
  IF the lump is empty  no textures ;  RETURN
  FOR EACH texture index i
    IF its data offset IS -1  CONTINUE               # not embedded
    IF width or height is not a multiple of 16
      FAIL WITH "Texture <name> is not 16 aligned"
    # Four mip levels of a w by h image total w*h*85/64 pixels:
    #   w*h + w*h/4 + w*h/16 + w*h/64 = w*h * 85/64
    pixels = width * height / 64 * 85
    allocate a header plus that many bytes
    copy the name, dimensions, and all four mip offsets, REBASED by the
         difference between the in-memory and on-disk header sizes
    copy the pixels immediately after the header
    IF the name begins with "sky"  initialize the sky texture from it

  # --- sequence the animations ---
  FOR EACH texture whose name begins with '+' and is not yet sequenced
    # The character AFTER the '+' selects the cycle and the frame:
    #   '0'..'9' is frame n of the PRIMARY cycle
    #   'a'..'j' (case-folded to 'A'..'J') is frame n of the ALTERNATE cycle
    # The rest of the name identifies the group.
    collect every texture sharing the name FROM THE THIRD CHARACTER ON
    into `anims[0..max-1]` and `altanims[0..altmax-1]` by that frame index
    FOR EACH collected frame j IN each cycle
      IF the slot is empty  FAIL WITH "Missing frame <j> of <name>"
      frame.anim_total = cycle length * 2               # tenths of a second
      frame.anim_min   = j * 2
      frame.anim_max   = (j+1) * 2
      frame.anim_next  = the next frame, WRAPPING            # a ring
      frame.alternate_anims = the OTHER cycle's first frame, if it exists
```

**Invariants** — five load-bearing details.

**The 85/64 pixel count** is the closed form of four mip levels, and it is why textures must be
multiples of 16: the fourth level is a sixteenth of the first in each dimension and must still be a
whole number of pixels.

**The mip offsets are rebased** because the in-memory header is larger than the on-disk one. A
rebuild storing offsets relative to the pixel data rather than the header avoids the adjustment.

**The animation cycle is discovered from names.** `+0lava1`, `+1lava1` and so on form a cycle; the
`a` through `j` variants form a second cycle for the same surface, and the two are cross-linked. The
second cycle is what a switch or a powered door toggles between — the renderer picks a cycle by the
entity's frame number ([`r_bsp.c`](r_bsp.c.md)). Up to ten frames per cycle.

**Each frame lasts two tenths of a second**, from the cycle constant, so a texture animation runs at
five frames per second. That is not configurable and every animated texture in the game assumes it.

**A missing frame is fatal.** A cycle declaring frames 0, 1 and 3 aborts the load.

Case folding means `+A` and `+a` are the same frame, so a cycle cannot have twenty frames.

A texture whose data offset is −1 is left absent and the texinfo loader substitutes a placeholder.

### `Mod_LoadTexinfo`

**Contract** — reads the texture-mapping array, byte-swapping the eight projection values,
classifying each surface's texture scale into one of four mip-adjustment levels, and resolving the
texture reference — substituting a checkerboard placeholder when the texture is missing and clearing
the flags in that case. An out-of-range texture index is a fatal error.

```text
FUNCTION mod_load_texinfo(lump)
  FOR EACH entry
    byte-swap the eight projection values
    # Classify by the average length of the two projection vectors.
    len = (length(vecs[0]) + length(vecs[1])) / 2
    mipadjust = 4 IF len < 0.32
                3 IF len < 0.49
                2 IF len < 0.99
                1 OTHERWISE
    miptex = little-endian ;  flags = little-endian
    IF the model has no textures at all
      texture = the placeholder ;  flags = 0
    ELSE
      IF miptex >= the texture count  FAIL WITH "miptex >= numtextures"
      texture = textures[miptex]
      IF it is absent  texture = the placeholder ;  flags = 0
```

**Invariants** — the **mip adjustment is a texture-scale compensation.** A surface whose texture is
stretched (short projection vectors) needs a coarser mip level at the same distance than one whose
texture is tiled tightly. The four thresholds quantize that into a multiplier the renderer applies
to its distance-based mip choice ([`r_surf.c`](r_surf.c.md)). A rebuild that omits it gets visibly
over-detailed mips on stretched surfaces.

The thresholds are stated as inequalities against 0.32, 0.49 and 0.99, which correspond to scales of
about 1/3, 1/2 and 1. The commented-out alternative computes a continuous reciprocal and is not what
ships.

Clearing the flags along with substituting the placeholder means a **missing sky or liquid texture
becomes an ordinary lit surface**, which is why a map with a missing texture shows a checkerboard
rather than failing.

### `CalcSurfaceExtents`

**Contract** — takes a surface; computes its extent in texture space, snapped outward to the 16-unit
lightmap grid. A non-special surface spanning more than 256 units in either direction is a fatal
error.

```text
FUNCTION calc_surface_extents(s)
  mins = (+999999, +999999) ;  maxs = (-99999, -99999)
  FOR EACH of the surface's edges
    e = surfedges[s.firstedge + i]
    v = the vertex at edges[|e|].v[0] IF e >= 0 ELSE edges[-e].v[1]
    FOR EACH texture axis j
      val = dot(v.position, vecs[j][0..2]) + vecs[j][3]
      widen mins[j] and maxs[j] BY val
  FOR EACH axis i
    s.texturemins[i] = floor(mins[i] / 16) * 16
    s.extents[i]     = (ceil(maxs[i] / 16) - floor(mins[i] / 16)) * 16
    IF the texture is not special AND s.extents[i] > 256
      FAIL WITH "Bad surface extents"
```

**Invariants** — the **256-unit cap** is the surface cache's block limit: a cached surface is at most
256 by 256 texels, so the map compiler must subdivide larger faces. That is why maps are built of
many small faces and why the compiler's subdivision size is 240 units. A special surface — sky or
liquid — is exempt because it has no cached surface.

The snapping to 16 is what makes the lightmap's dimensions derivable: a lightmap is
`extents/16 + 1` samples on each axis.

The edge traversal honours the negative-index convention, taking the *first* vertex of a forward
edge and the *second* of a reversed one — which is what makes the loop visit each vertex once
around the polygon.

**Notes** — the two initial values are asymmetric by a factor of ten (999999 against −99999), which
is a typographical slip. Harmless: every surface has vertices.

### `Mod_LoadFaces`

**Contract** — reads the face array, resolving each face's plane, texture mapping, lightmap offset
and light styles; computes its texture extents; and sets its drawing flags from its texture's name.

```text
FUNCTION mod_load_faces(lump)
  FOR EACH face
    firstedge, numedges from the file
    flags = 0
    IF the file's side flag is set  set the plane-back flag
    plane = planes[the file's plane number]
    texinfo = texinfo[the file's texinfo number]
    calc_surface_extents(face)
    styles = the file's four style bytes
    samples = lightdata + the file's offset, OR absent when the offset IS -1

    IF the texture's name begins with "sky"
      set the draw-sky AND draw-tiled flags ;  CONTINUE
    IF the texture's name begins with "*"                 # a liquid
      set the draw-turbulent AND draw-tiled flags
      # Override the extents so the warp can wrap freely:
      extents = (16384, 16384) ;  texturemins = (-8192, -8192)
      CONTINUE
```

**Invariants** — the drawing flags are set **from the texture's name**, which is the third and last
place this engine dispatches on a string in a binary format. Sky and liquid are recognized by
prefix.

**A liquid surface's extents are overwritten with an enormous range.** The sine warp
([`d_scan.c`](d_scan.c.md)) samples texture coordinates far outside the surface's real extent, and
the extents are used to bound the sampling; setting them to ±8192 lets the warp wrap. This is a real
requirement, not a hack to tidy: without it liquid surfaces show a seam where the warp leaves the
extent.

Both special cases set the **tiled** flag, which is what routes them past the surface cache
entirely.

### `Mod_LoadNodes`, `Mod_SetParent`

**Contract** — read the interior nodes, converting each child index into a pointer — into the node
array for a non-negative index and into the leaf array for a negative one — then walk the tree
setting every node's and leaf's parent.

```text
FUNCTION mod_load_nodes(lump)
  FOR EACH node
    minmaxs = the six 16-bit bounds
    plane = planes[the file's plane number]
    firstsurface, numsurfaces from the file
    FOR EACH child j
      p = the file's child index
      children[j] = nodes[p]        IF p >= 0
                    leafs[-1 - p]   OTHERWISE
  mod_set_parent(the root, nothing)

FUNCTION mod_set_parent(node, parent)
  node.parent = parent
  IF node is a leaf  RETURN
  mod_set_parent(node.children[0], node)
  mod_set_parent(node.children[1], node)
```

**Invariants** — the negative-child decoding is `-1 - p`, which maps −1 to leaf 0 and −2 to leaf 1.
That matches [`bspfile.h`](bspfile.h.md)'s `-(leaf+1)` encoding.

**Parent pointers are added at load and are not in the file.** They let the renderer walk *upward*
from a visible leaf marking the chain of nodes that must be traversed
([`r_bsp.c`](r_bsp.c.md)), which is the trick that makes the drawing walk visit only
visible parts of the tree.

The node's contents field is never assigned and therefore stays zero from the hunk's zero-fill,
which is exactly what the node-versus-leaf discriminant requires. A rebuild must set it explicitly.

### `Mod_LoadLeafs`

**Contract** — reads the leaves, resolving each one's mark-surface range, visibility offset and
ambient levels, and clearing its entity-fragment list.

**Invariants** — a visibility offset of −1 leaves the pointer absent, which
[`Mod_LeafPVS`](#mod_leafpvs) turns into an all-visible answer.

### `Mod_LoadClipnodes`

**Contract** — reads the collision nodes **as-is**, without converting indices to pointers, and
establishes two of the four hulls over them with hard-coded body sizes.

```text
FUNCTION mod_load_clipnodes(lump)
  copy the nodes, byte-swapping the plane number and both children
  # Hull 1 — the PLAYER and small monsters:
  hulls[1] spans the whole node array
  hulls[1].clip_mins = (-16, -16, -24)
  hulls[1].clip_maxs = ( 16,  16,  32)
  # Hull 2 — LARGE monsters:
  hulls[2] spans the whole node array
  hulls[2].clip_mins = (-32, -32, -24)
  hulls[2].clip_maxs = ( 32,  32,  64)
```

**Invariants** — **these six numbers are the most important unwritten contract in the tree.** The
map compiler shrank the map's geometry by exactly these boxes to produce the two collision trees, and
the engine must use the same values to compute its trace offsets
([`world.c`](world.c.md#sv_hullforentity)). They appear nowhere in the map file. A rebuild that
changes them, or that guesses them, produces collision that is subtly wrong everywhere — players
floating above floors or sinking into them.

The player's box is **56 units tall with its origin 24 units above the bottom**, which is the eye
height convention the whole game is built around.

Both hulls index the same node array, distinguished only by their root, which the sub-model loop
sets per sub-model.

The nodes keep their **on-disk form** — index-based children, negative values meaning contents —
because [`world.c`](world.c.md) walks them directly. So the collision tree is the one structure the
loader does not translate.

### `Mod_MakeHull0`

**Contract** — builds a fourth hull from the rendering tree, converting each node's plane pointer
back to an index and each child pointer back to either a node index or a contents value.

```text
FUNCTION mod_make_hull0()
  allocate one collision node per rendering node
  hulls[0] spans them ;  planes = the model's planes
  FOR EACH rendering node
    out.planenum = the index OF in.plane
    FOR EACH child j
      out.children[j] = child.contents         IF the child is a LEAF
                        the index OF the child  OTHERWISE
```

**Invariants** — this is a *point* collision tree, built from the draw tree, so a point trace through
it reports the real contents at a point rather than the shrunken player-sized approximation. That is
why [`SV_PointContents`](world.c.md#sv_pointcontents-sv_truepointcontents) uses hull 0 and why liquid detection is
exact while player collision is not.

The conversion is the inverse of what [`Mod_LoadNodes`](#mod_loadnodes-mod_setparent) just did, which is why the
rendering tree is loaded first.

### `RadiusFromBounds`

**Contract** — returns the distance from the origin to the furthest corner of a box, taking the
larger magnitude on each axis.

**Invariants** — the radius is about the **model's own origin**, not the box's centre, which is
correct for a rotating sub-model.

## Loading an animated model

### `Mod_LoadAliasModel`

**Contract** — reads an animated model. Validates the version, allocates one hunk block for the
header, the model record, the texture coordinates and the triangles; validates and byte-swaps the
header; loads the skins, the texture coordinates (converted to fixed point), the triangles and the
frames; then **copies the entire result into the evictable cache and frees the hunk**.

```text
FUNCTION mod_load_alias_model(mod, buffer)
  start = the hunk's low mark
  IF the version is not 6  FAIL WITH "wrong version number"

  # One block holds the header (with its variable frame table), the model
  # record, the texture coordinates and the triangles. Skins and frame data
  # are allocated separately and referenced by OFFSET.
  allocate header + (numframes-1) frame descriptors + model + stverts + triangles

  mod.flags = the file's flag word         # the effect flags of model.h
  byte-swap and validate the header:
    IF the skin is taller than the maximum image height  FAIL
    IF numverts <= 0                                     FAIL "no vertices"
    IF numverts > the vertex maximum                     FAIL "too many vertices"
    IF numtris  <= 0                                     FAIL "no triangles"
    IF the skin width is not a multiple of 4             FAIL
    IF numskins  < 1                                     FAIL
    IF numframes < 1                                     FAIL
  the model's average-size field is scaled by a fixed ratio

  FOR EACH skin: load it as a single image or as a group
  FOR EACH texture coordinate: copy the seam flag, and shift s and t LEFT 16
                               so they become 16.16 fixed point
  FOR EACH triangle: copy the front-facing flag and the three vertex indices
  FOR EACH frame: load it as a single frame or as a group

  mod.type = alias
  mod.mins = (-16,-16,-16) ;  mod.maxs = (16,16,16)      # see the note

  # Relocate the whole thing into the cache.
  total = the hunk's low mark - start
  cache_alloc(mod.cache, total)
  IF the allocation produced nothing  RETURN
  copy `total` bytes FROM the header TO the cache
  restore the hunk to `start`
```

**Invariants** — the **build-then-relocate** pattern is the heart of it. The model is assembled on
the hunk, where allocations are cheap and contiguous, and then copied as one block into the cache.
The copy is why **every internal reference must be an offset** ([`model.h`](model.h.md)): the block
moves, and an offset survives the move while a pointer does not.

The copy relies on every allocation between the two marks belonging to this model and being
contiguous — which is true because the hunk is a stack and nothing else allocates during the load.
A rebuild that interleaves allocations breaks it.

**Texture coordinates are converted to 16.16 fixed point at load** by a left shift of 16. The
software rasterizer steps them in that format ([`r_alias.c`](r_alias.c.md)), so the conversion
belongs at load rather than per frame.

**The bounding box is hard-coded to ±16 units** and the source marks it as wrong. So every animated
model in the game has the same collision and culling box regardless of its real size, which is why a
large monster can be culled while still partly visible. A rebuild should compute the box from the
frames' own bounds, which are loaded and otherwise unused — but doing so changes culling and
therefore what is drawn.

A failed cache allocation returns **without freeing the hunk**, leaking the whole model until the
level changes.

### `Mod_LoadAliasFrame`, `Mod_LoadAliasGroup`

**Contract** — `Mod_LoadAliasFrame` copies one frame's name, quantized bounding box and vertex array,
recording the array's offset. `Mod_LoadAliasGroup` reads a frame count, allocates a group, copies its
bounding box and per-frame intervals, and then loads each frame through the first. Both return the
position after what they consumed. A non-positive interval is a fatal error.

**Invariants** — the vertex data needs **no byte swapping at all**, because every value in it is a
single byte — which is the compression's second benefit and is noted in the source.

Intervals are cumulative end times from the group's start
([`modelgen.h`](modelgen.h.md)), and requiring them positive is what makes the playback search
terminate.

### `Mod_LoadAliasSkin`, `Mod_LoadAliasSkinGroup`

**Contract** — copy one skin's pixels, converting to the renderer's destination pixel width — a
straight copy at one byte per pixel, a palette lookup at two. Any other width is a fatal error. The
group form reads a count and intervals first, as with frames.

**Invariants** — **the skin is converted to the display's pixel format at load time**, which means
a model loaded while the display is 8-bit is wrong if the display later becomes 16-bit. The engine
avoids it by reloading everything on a mode change
([`host.c`](host.c.md#host_clearmemory)), which is why changing resolution reloads the level.

## Loading a sprite

### `Mod_LoadSpriteModel`

**Contract** — reads a sprite. Validates the version, allocates a record with a variable frame
table, copies the orientation rule and dimensions, derives the model's bounding box from half its
maximum dimensions, and loads each frame as a single image or a group. A frame count below one is a
fatal error.

**Invariants** — the record is allocated on the **hunk** and its address is then assigned directly
into the cache handle. So a sprite is reached through a cache handle but does not live in the
cache — which is the anomaly [`Mod_ClearAll`](#mod_clearall) has to work around. A rebuild should
either put sprites in the cache properly or give them their own accessor.

The bounding box is **half the maximum frame's dimensions in each direction**, centred — which is
right for a billboard and wrong for a sprite with an off-centre origin.

### `Mod_LoadSpriteFrame`, `Mod_LoadSpriteGroup`

**Contract** — `Mod_LoadSpriteFrame` reads a frame's dimensions and pixel origin, converts the
origin into four signed edge distances, and copies the pixels in the display's format.
`Mod_LoadSpriteGroup` reads a count and per-frame intervals, then loads each frame.

```text
FUNCTION mod_load_sprite_frame(pin) -> MSpriteFrame
  width, height from the file ;  origin from the file
  up    = origin[1]
  down  = origin[1] - height
  left  = origin[0]
  right = origin[0] + width
  copy the pixels, expanding through the palette when the display is 16-bit
```

**Invariants** — the four edge distances are the conversion
[`model.h`](model.h.md) describes. The vertical pair is derived by *subtracting* the height from the
origin, so a conventionally centred sprite has a positive `up` and a negative `down`.

**Notes** — the record is zero-filled using the *unconverted* size, so at two bytes per pixel only
half the pixel area is cleared before being overwritten. Harmless.
