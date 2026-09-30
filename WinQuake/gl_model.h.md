# WinQuake/gl_model.h

> The model records as the hardware renderer needs them: surfaces carry pre-built vertex lists and a lightmap page coordinate instead of a cache slot, and textures carry a library handle instead of pixels.

**Needs** — [`modelgen.h`](modelgen.h.md) · [`spritegn.h`](spritegn.h.md) · [`bspfile.h`](bspfile.h.md)
**Used by** — every `gl_*` twin, in place of [`model.h`](model.h.md)
**Tier floor** — none

## Purpose

The same world, brush, sprite and animated-model records as [`model.h`](model.h.md) — read that twin for the
structure, the three collision hulls and the visibility encoding, all of which are unchanged. This twin records only
the differences, because the differences are precisely the design of the hardware renderer.

## State

The records and constants this header declares are the state; they are described in the sections below.

## What differs, and why each difference exists

**A surface owns a list of ready-to-submit polygons.** Where the software renderer walks a surface's edge list every
frame ([`r_bsp.c`](r_bsp.c.md)), here the loader converts each surface once into one or more vertex lists, each vertex
carrying a position, a texture coordinate **and a second texture coordinate for the lightmap**. Seven numbers per
vertex, four vertices minimum, variable beyond. This is the central change: geometry is prepared at load time because
the hardware wants buffers, not spans.

There may be several polygons per surface, because a water or sky surface is subdivided
([`gl_warp.c`](gl_warp.c.md)) so that a per-vertex distortion looks smooth.

**A surface records where its lightmap lives in a page**, as a coordinate pair plus a page number, replacing the
software renderer's array of cache slots per mip level. The lit surface is no longer generated on demand and cached —
it is a rectangle in one of a few large textures, uploaded once and patched when a light changes
([`gl_rsurf.c`](gl_rsurf.c.md)). So the whole surface-cache machinery
([`d_surf.c`](d_surf.c.md), [`r_surf.c`](r_surf.c.md)) has no counterpart here, and neither does its
thrash detection.

**A surface records the light levels currently baked into its page rectangle**, plus whether a dynamic light is in
there. That is the invalidation test, and it is the same test the software cache made
([`r_surf.c`](r_surf.c.md)) — moved from "is this cache entry still valid" to "must I re-upload this rectangle".

**A surface carries a chain pointer to the next surface with the same texture.** Draw order is by texture, not by
depth or by tree order, because changing texture was the expensive operation. The software renderer had no such need
and no such field.

**A texture carries a library handle, not pixels**, and the mip chain is the library's problem. A brush texture also
carries its own surface chain, for the same reason.

**Bounding boxes are real numbers rather than small integers**, since nothing here exploits integer comparison the way
the span sorter did.

**A surface can be flagged as underwater**, which selects the distorting path at draw time.

**An animated model's frame group records its first pose, its pose count and its interval** directly, rather than a
tagged union of single and grouped frames. The loader flattens the two cases, so the renderer has one path. That
flattening is worth copying: the tagged form in [`model.h`](model.h.md) buys nothing and costs a branch in the inner
loop.

**Skin records lose their cache slot and their tagged form** for the same reason: a skin is uploaded once and is a
handle thereafter.

**Notes** — an entity-effect set (bright field, muzzle flash, bright light, dim light) is declared here rather than in
the protocol header. That is a misplacement, not a decision; a rebuild declares it once beside the protocol
([`protocol.h`](protocol.h.md)).

The two headers being separate files with ninety percent shared content is the source's answer to a problem a rebuild
should solve differently: one record set with the renderer-specific fields factored out, or two renderers over one
neutral model. The recipe records the split because the duplication is real and a reader comparing twins needs to know
which header a given `gl_*` file sees.
