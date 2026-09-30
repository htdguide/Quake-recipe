# WinQuake/gl_mesh.c

> Converts an animated model's triangle soup, once at load, into the longest strips and fans it can find, producing one command list and one vertex order that every pose of that model shares.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md) · [`gl_model.h`](gl_model.h.md) · [`modelgen.h`](modelgen.h.md)
**Used by** — [`gl_model.c`](gl_model.c.md) calls it at the end of loading an animated model
**Tier floor** — none

## Purpose

The one file in the hardware renderer with no software counterpart at all. Models in the file format
([`modelgen.h`](modelgen.h.md)) are an unordered list of triangles over a shared vertex list; the hardware wants long
runs of connected triangles. This converts one to the other, exploiting the fact that **the topology is identical in
every pose**, so the conversion is done once per model and reused for every frame of every animation.

That reuse is what makes [`GL_DrawAliasFrame`](gl_rmain.c.md#gl_drawaliasframe) a single pass over a precomputed list.

## State

```text
VARIABLE commands   : int[8192]       # the output: counts, coordinates, terminator
VARIABLE numcommands
VARIABLE vertexorder : int[8192]      # the output: which source vertex each slot uses
VARIABLE numorder
VARIABLE used : byte[8192]            # 0 = free, 1 = consumed, 2 = tentative
VARIABLE stripverts, striptris : int[128] ;  stripcount
```

**Invariants** — the marker has **three states, not two**: free, permanently consumed, and *tentatively* taken by the run
currently being measured. Both search functions mark tentatively as they extend and clear every tentative mark before
returning, because they are called speculatively — several runs are measured from one triangle and only the best is
kept. Collapsing this to a boolean makes the search consume triangles it merely considered.

Everything is a fixed-size array with no bound check. A rebuild sizes from the model.

## `StripLength`, `FanLength`

**Contract** — each takes a starting triangle and which of its three corners to start at, and extends a run as far as it
can by repeatedly scanning for an unconsumed triangle sharing the run's current open edge and facing the same way.
Returns the run's length, leaving the vertex sequence and the triangle list in the shared buffers.

```text
FUNCTION strip_length(start_tri, start_corner) -> int
  mark start_tri tentative
  the run's first three vertices are start_tri's corners, rotated to start_corner
  # the open edge, as an ordered pair
  m1 = corner start+2 ;  m2 = corner start+1          # STRIP
  # for a fan:  m1 = corner start+0 ;  m2 = corner start+2
  LOOP
    FOR EACH later triangle
      SKIP if its facing-forward flag differs from the run's
      FOR EACH of its three rotations
        IF its first two vertices are exactly (m1, m2)
          IF it is already consumed  the run ends here
          append its third vertex to the run
          STRIP: alternate which of m1/m2 the new vertex replaces
          FAN:   always replace m2, so m1 stays the fan's hub
          mark it tentative ;  continue the outer LOOP
    the run ends
  clear every tentative mark except the start's
  RETURN the length
```

**Invariants** —

- **The matching edge is directed**, tested as an ordered pair, so only a triangle with consistent winding joins the
  run. That is what keeps the output's facing correct without any winding fix-up.
- The two functions differ in **one line**: a strip alternates which endpoint of the open edge advances, a fan never
  advances the hub. That is precisely the difference between the two primitives, and writing them as one function with a
  flag is the obvious rebuild simplification.
- A triangle's **facing-forward flag** ([`modelgen.h`](modelgen.h.md)) must match across a run, because it selects which
  of two texture coordinate sets the vertex uses — the format stores seam vertices once with a front and a back
  coordinate. Mixing them in one run would give a vertex two coordinates in one primitive, which no primitive allows.
- The search is a **linear scan over all later triangles**, per extension step — so building a model is quadratic in its
  triangle count. At load time for a few-hundred-triangle model that is free. A rebuild with an edge-to-triangle map
  makes it linear and should, but the recipe records that it does not need to.
- The scan starts *after* the starting triangle, so runs never extend backwards. That loses some length and is why the
  outer loop below tries every starting corner.

## `BuildTris`

**Contract** — greedily covers the model with runs: for each unconsumed triangle, measure a strip and a fan from each of
its three corners, keep the longest, consume its triangles, and append it to the output as a signed count, the texture
coordinates of each of its vertices, and the source vertex index of each. Terminates the command list with a zero.

```text
FUNCTION build_tris()
  clear used, commands, vertexorder
  FOR EACH triangle i
    IF used[i]  SKIP
    best = 0
    FOR EACH corner c of 3
      len = strip_length(i, c) ;  IF len > best  best = len, kind = strip, remember the run
      len = fan_length(i, c)   ;  IF len > best  best = len, kind = fan,   remember the run
    mark every triangle of the best run consumed
    emit best (NEGATED if it is a fan) into the command list
    FOR EACH vertex of the run
      look up its texture coordinate, using the back-seam offset when the
        triangle does not face forward
      nudge both coordinates by half a texel and normalize by the skin size
      emit the two coordinates into the command list
      emit the source vertex index into the vertex order
  emit 0 to terminate the command list
```

**Invariants** —

- **The sign of the count carries the primitive kind**: negative for a fan, positive for a strip, zero to stop. One
  integer, three meanings — and the draw loop ([`gl_rmain.c`](gl_rmain.c.md#gl_drawaliasframe)) reads it exactly that
  way.
- **Texture coordinates go in the command list; positions do not.** The command list is shared by every pose. Positions
  are looked up per pose through the vertex order, which is why a separate vertex-order array exists and why every pose's
  vertices are afterwards *rewritten* into that order by the caller
  ([`gl_model.c`](gl_model.c.md)) — so the draw loop walks poses linearly with no indirection at all.
- Each texture coordinate is nudged by **half a texel** before normalizing, centring the sample in its texel. Same
  correction as the lightmap coordinates in [`gl_rsurf.c`](gl_rsurf.c.md), same symptom if omitted.
- The **back-seam offset** — half the skin width added to the horizontal coordinate — is how the format stores a vertex
  that appears on both sides of a model's texture seam only once. The facing flag selects which copy. A rebuild reading
  this format must apply it or seams tear.
- The greedy choice is **longest-first among six candidates per triangle**, not optimal. Optimal strip decomposition is
  hard and the gain is small; recorded so a rebuilder does not go looking for a cleverer algorithm they are expected to
  reproduce.

## `GL_MakeAliasModelDisplayLists`

**Contract** — builds the runs for one model, optionally caching the result to a file beside the model, then allocates the
command list and the reordered pose data on the load heap and stores them in the model. Every pose's vertices are copied
into the vertex order the command list expects.

**Invariants** — the reordering **expands the vertex data**, because a vertex used by three runs is stored three times.
That is the cost of removing the indirection, and it is paid once per model at load. A rebuild using indexed drawing keeps
the source order and the index list instead, which is smaller and equally fast on modern hardware — the recipe records the
original's choice and the reason it made sense on hardware with no index support worth using.

The optional cache file is a build-time convenience, not part of the format, and a rebuild can ignore it.
