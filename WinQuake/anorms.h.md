# WinQuake/anorms.h

> The fixed table of 162 unit normals that every animated model's vertices index instead of carrying normals of their own.

**Needs** — nothing; a table of literals
**Used by** — [`r_alias.c`](r_alias.c.md) · [`gl_mesh.c`](gl_mesh.c.md) · [`gl_rmain.c`](gl_rmain.c.md) · [`anorm_dots.h`](anorm_dots.h.md) is derived from it
**Tier floor** — none

## Purpose

A model vertex carries one byte naming its normal ([`modelgen.h`](modelgen.h.md)). This is the table
that byte indexes, and it is **part of the model format, not an implementation choice**: every published
model's indices refer to these 162 directions by position.

## State

```text
CONSTANT bytedirs : vec3[162]      # unit vectors, an icosahedral subdivision of
                                   # the sphere
```

**Invariants** — 162 entries, so a one-byte index has 94 unused values. The directions are the vertices
of a twice-subdivided icosahedron, which is why they are near-uniformly spaced and why the values look
like combinations of 0.525731, 0.850651 and 0.309017 — the golden-ratio constants of an icosahedron.

The table's **order is arbitrary and load-bearing.** A rebuild cannot regenerate it from the geometry,
because the model compiler chose these indices and published models use them. The values must be
copied.

The quantization to 162 directions is about 14 degrees of angular resolution, which is coarse — and it
is why lighting on a model in this game is flat and slightly faceted rather than smooth.

**Notes** — the table's precision is six decimal places, which is more than a single-precision float
holds. A rebuild should copy the literals as written; rounding them differently shifts the derived
brightness table ([`anorm_dots.h`](anorm_dots.h.md)) and produces visibly different model shading.
