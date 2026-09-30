# WinQuake/gl_model.c

> The map and model loader for the hardware renderer: the same file formats and the same three collision hulls, with every surface turned into vertex lists and every texture uploaded at load time instead of kept as pixels.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`gl_model.h`](gl_model.h.md) · [`glquake.h`](glquake.h.md) · [`bspfile.h`](bspfile.h.md) · [`modelgen.h`](modelgen.h.md) · [`spritegn.h`](spritegn.h.md) · [`gl_warp.c`](gl_warp.c.md) · [`gl_mesh.c`](gl_mesh.c.md) · [`gl_draw.c`](gl_draw.c.md)
**Used by** — [`cl_parse.c`](cl_parse.c.md) · [`gl_rsurf.c`](gl_rsurf.c.md) · [`sv_main.c`](sv_main.c.md) · [`world.c`](world.c.md)
**Tier floor** — none

## Purpose

Read [`model.c`](model.c.md) first and in full. Every file format, every record, the visibility decompression, the
three precompiled collision hulls at their **hard-coded body sizes**, the hull-zero fabrication, the surface extent
calculation, the animation group naming convention and the model cache all carry over unchanged, and they are the
load-bearing content. This twin records only what the hardware renderer needed to change.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## What differs from the software loader

**Every texture is uploaded as it is read and only its handle is kept.** Map textures, model skins and sprite frames all
go through the upload path in [`gl_draw.c`](gl_draw.c.md) under a name derived from the map or model, so the registry
deduplicates them. The software loader instead keeps the mip chain in memory for the surface cache to sample; here the
pixels are not needed again — with two exceptions:

- The **sky texture** is split in half and uploaded as two layers ([`gl_warp.c`](gl_warp.c.md)).
- A **player model's skin** is kept as pixels as well, because it is recoloured per player
  ([`gl_rmisc.c`](gl_rmisc.c.md)).

**Every surface gets a vertex list built for it.** After a surface's extents are computed, it is either subdivided — if
it is sky or liquid ([`gl_warp.c`](gl_warp.c.md)) — or converted directly into one vertex list
([`gl_rsurf.c`](gl_rsurf.c.md#buildsurfacedisplaylist)). Nothing walks a surface's edge list at draw time.

**Every surface gets a rectangle in a lightmap page**, allocated and filled after all surfaces are loaded, since the
packer wants to see them in order ([`gl_rsurf.c`](gl_rsurf.c.md#gl_createsurfacelightmap-gl_buildlightmaps)).

**Surfaces are flagged as underwater at load**, by checking the texture name's prefix, so the draw path needs no string
comparison.

**Animated model poses are flattened and reordered.** The format's distinction between a single frame and a timed group
collapses into a first-pose index, a count and an interval ([`gl_model.h`](gl_model.h.md)); then the whole pose set is
rewritten into the vertex order the strip builder produced ([`gl_mesh.c`](gl_mesh.c.md)), so the draw loop is a linear
walk.

**Model skins are flood-filled before upload.** The transparent index in a skin is replaced by the nearest opaque colour,
spreading outward from the opaque pixels.

```text
FUNCTION flood_fill_skin(pixels, width, height)
  fill_colour = the pixel at the top-left corner
  IF fill_colour is not the transparent index  RETURN
  seed a queue with the corner
  WHILE the queue is not empty
    take a pixel; for each of its four neighbours
      IF the neighbour is the fill colour  mark it and enqueue it
      ELSE remember the neighbour's colour as the replacement
    write the replacement over the pixel
```

**Invariants** — this exists because **the hardware filters between texels and the transparent index is a real colour**.
Filtering a skin's edge against its transparent background would fringe every model with that colour. Bleeding the
adjacent opaque colour outward means the filter has something correct to blend with. The software renderer point-samples
and needs none of this.

This is one of the least obvious requirements in the whole hardware port, and a rebuild that filters its textures needs
it — for model skins and for any alpha-tested image.

**Bounding boxes are stored as real numbers** rather than small integers, and a bounding radius is computed per brush
model for the frustum rejection in [`gl_rsurf.c`](gl_rsurf.c.md#r_drawbrushmodel).

**The texture animation chains are built the same way** — by name convention, two groups of up to ten frames — because
the convention is part of the content, not of the renderer.

**Notes** — the two loaders being separate near-duplicate files is the source's cost for supporting two renderers.
A rebuild should load once into a renderer-neutral model and let the renderer attach its own per-surface preparation,
which is the difference [`gl_model.h`](gl_model.h.md) already describes.
