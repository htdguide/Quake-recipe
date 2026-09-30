# QW/server/model.c

> The map loader for the dedicated server: the collision hulls, the leaf structure and the visibility data, with everything the renderer would have needed left out.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`model.h`](../client/model.h.md) · [`bspfile.h`](../client/bspfile.h.md) · [`crc.h`](../client/crc.h.md)
**Used by** — [`sv_init.c`](sv_init.c.md) · [`world.c`](world.c.md) · [`sv_ents.c`](sv_ents.c.md)
**Tier floor** — none

## Purpose

Read [`model.c`](../../WinQuake/model.c.md) for the file formats, the three precompiled collision hulls at their hard-coded body
sizes, the hull-zero fabrication, the visibility decompression and the model cache. All of it is here and unchanged.

## State

As [`model.c`](../../WinQuake/model.c.md); the records are unchanged except where **What differs** says otherwise.

## What differs

**Almost half the file is gone**, and the deletions are the whole answer to "what does a server not need from a map":

- **No surfaces, no texture information, no lightmaps, no light data.** The server never draws, so faces, texture coordinates,
  surface extents and the lightmap samples are not read at all.
- **No animated models and no sprites.** The server needs a model's *bounding box* to collide with it, which it obtains from a
  cheap header read, and nothing else. So the frame data, the skins, the triangle lists and the whole sprite loader are absent.
- **No edge or vertex arrays beyond what the hulls need.**

What remains is: the planes, the nodes, the leaves, the leaf-to-face marks (needed only for the leaf structure's shape), the three
clipping-node hulls, the submodel table, the visibility data, and the entity text.

**Invariants** —

- **The three collision hulls and their hard-coded body sizes are load-bearing and unchanged.** They are the unwritten contract
  with the map compiler ([`model.c`](../../WinQuake/model.c.md)), and the movement model's fixed player box
  ([`pmove.c`](../client/pmove.c.md)) is the same size.
- **The visibility data is loaded but not decompressed here**; the server decompresses it whole and derives the hearable set at
  map load instead ([`sv_init.c`](sv_init.c.md)).
- **A model's checksum is computed while loading** for the anti-modification comparison
  ([`sv_init.c`](sv_init.c.md)).

**Notes** — the measurement worth keeping: **a server needs roughly half a map file and no model files at all.** For a rebuild
that wants a lightweight headless server, this file is the inventory of what to parse and what to skip, and it is more useful than
reading the format specification.
