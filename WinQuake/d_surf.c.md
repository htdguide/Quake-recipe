# WinQuake/d_surf.c

> The surface cache: a rover-allocated ring holding lit, mipped copies of surfaces, with an invalidation rule keyed on light levels and texture animation, and a thrashing detector that tells the player when it is too small.

**Needs** — [`d_local.h`](d_local.h.md) · [`r_local.h`](r_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [`common.h`](common.h.md) · [`console.h`](console.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`d_edge.c`](d_edge.c.md) requests surfaces; [`r_surf.c`](r_surf.c.md) fills them; the video backends size and allocate the cache ([`render.h`](render.h.md#surface-cache))
**Tier floor** — T1 as written; a T2 rebuild with an index-based free list meets every contract

## Purpose

The software renderer does not light pixels as it draws them. It builds a **lit, mipped copy of each
visible surface** once, caches it, and then the span fillers merely sample it. That is why the inner loop
is a texture fetch with no lighting arithmetic
([`d_scan.c`](d_scan.c.md)), and it is the second-largest performance decision in the engine after the
edge sorter.

The cache is a **rover ring**: allocation walks forward from wherever the last one ended, coalescing and
evicting whatever it passes. There is no least-recently-used ordering and no reference counting. The
consequence is that a frame needing more surface area than the cache holds will evict a surface it has
already drawn and need again — thrashing — and the file detects and reports exactly that.

## State

```text
VARIABLE sc_base  : SurfCache        # the cache's start
VARIABLE sc_size  : int              # its usable size, excluding the guard
VARIABLE sc_rover : SurfCache        # the allocation cursor
VARIABLE surfscale : real            # one over two to the mip level
VARIABLE r_cache_thrash : bool       # reported to the player
CONSTANT guardsize = 4
```

**Invariants** — entries tile the cache with no gaps, linked in **address order**, and a null owner marks a
free entry ([`d_local.h`](d_local.h.md)). The rover may point at any entry.

## Sizing

### `D_SurfaceCacheForRes`

**Contract** — takes a resolution; returns the bytes the cache needs. A command-line override in kilobytes
wins outright.

```text
FUNCTION d_surface_cache_for_res(width, height) -> int
  IF "-surfcachesize" IS present  RETURN its value * 1024
  size = 614400                                # 600 KB at 320x200
  pixels = width * height
  IF pixels > 64000  size = size + (pixels - 64000) * 3
  RETURN size
```

**Invariants** — **three bytes of cache per pixel above the base resolution.** The derivation: a surface is
cached at roughly the screen resolution it is drawn at, so the working set grows with pixel count; the
factor of three is empirical headroom for surfaces cached at more than one mip level and for fragmentation.

At 640 by 480 that is about 1.5 megabytes, which on an eight-megabyte budget
([`zone.h`](zone.h.md)) is why high resolutions were memory-constrained on the original hardware.

This function is called by the **video backend**, which then reserves the memory from the high end of the
hunk ([`render.h`](render.h.md#surface-cache)) — the inversion that exists because only the backend knows
the resolution.

### `D_InitCaches`

**Contract** — takes a buffer and a size; formats it as one free entry and installs the guard bytes. Prints
the size unless suppressed.

### `D_CheckCacheGuard`, `D_ClearCacheGuard`

**Contract** — write four bytes of a known pattern just past the cache and verify them. A mismatch is a
fatal error.

**Invariants** — checked on **every allocation**, unconditionally. An overrun of the surface cache is
otherwise silent and corrupts whatever the video backend put next to it, so the guard is the only thing
that turns that into a diagnosable failure. A rebuild should keep it, or should bound-check properly.

### `D_FlushCaches`

**Contract** — clears every entry's owner pointer, so every surface learns its cached form is gone, and
reformats the whole cache as one free entry.

**Invariants** — called at a level change and by the console's flush command. Walking the address-ordered
list and clearing each owner is what makes the invalidation complete — without it, a surface record from
the previous level would still point into the cache.

## Allocation

### `D_SCAlloc`

**Contract** — takes a width and a payload size; returns a free entry of at least that size, evicting and
coalescing forward from the rover. Wraps to the start when the remaining space is insufficient. A width
outside 0 to 256, a size outside 1 to 65536, or a size exceeding the whole cache is a fatal error.
Reaching the end of the list while still short is a fatal error.

```text
FUNCTION d_sc_alloc(width, size) -> SurfCache
  IF width outside 0..256   FAIL WITH "bad cache width"
  IF size outside 1..65536  FAIL WITH "bad cache size"
  size = round_up_to_multiple_of_4(size + the header's size)
  IF size > sc_size  FAIL WITH "<size> > cache size"

  wrapped = false
  IF the rover is unset OR there is not `size` after it
    IF the rover was set  wrapped = true
    sc_rover = sc_base                        # start over from the beginning

  # Evict and coalesce forward until the entry is large enough.
  new = sc_rover
  IF new has an owner  clear that owner        # tell the surface it is gone
  WHILE new.size < size
    sc_rover = sc_rover.next
    IF there is none  FAIL WITH "hit the end of memory"
    IF sc_rover has an owner  clear it
    new.size = new.size + sc_rover.size       # absorb it
    new.next = sc_rover.next

  # Split off a remainder only if it is worth a header.
  IF new.size - size > 256
    sc_rover = new + size
    sc_rover.size = new.size - size ;  sc_rover.next = new.next
    sc_rover.width = 0 ;  sc_rover.owner = nothing
    new.next = sc_rover ;  new.size = size
  ELSE
    sc_rover = new.next                       # the remainder is wasted

  new.width = width
  IF width > 0  new.height = the payload size divided by width
  new.owner = nothing                         # the caller sets it

  # --- thrashing detection ---
  IF d_roverwrapped
    IF wrapped OR sc_rover >= d_initial_rover  r_cache_thrash = true
  ELSE IF wrapped
    d_roverwrapped = true

  d_check_cache_guard()
  RETURN new
```

**Invariants** — five load-bearing decisions.

**Eviction is by position, not by age.** Whatever the rover passes is thrown away regardless of how
recently it was used. That is the cheapest possible policy and it is why thrashing is possible at all; the
compensation is that the rover advances monotonically, so within one frame each surface is evicted at most
once.

**The split threshold is 256 bytes.** A remainder smaller than that is left inside the allocation and
wasted, because a header plus a payload too small to hold any surface is worse than the waste.

**Thrashing is detected in two stages.** The first full wrap within a frame merely sets a flag; a *second*
wrap, or a rover that has passed the frame's starting position, means the cache has come all the way round
and is now evicting surfaces this frame already built. That is genuine thrashing and it is reported to the
player by flashing the screen edge ([`screen.c`](screen.c.md)). The two-stage test is what avoids
false positives on a frame that happens to wrap once near the end.

**The width bound of 256 is the surface extent limit** enforced at map load
([`model.c`](model.c.md#calcsurfaceextents)), and the size bound of 65536 is 256 squared. So the two
validations here are restating the map format's constraint.

**The height is computed only for diagnostics**, as the comment notes.

### `D_SCDump`

**Contract** — prints every entry with its size and width, marking the rover. A diagnostic.

## Helpers

### `MaskForNum`, `D_log2`

**Contract** — `MaskForNum` returns one less than a power-of-two dimension, or 255 for anything else, with
the comment explaining the rule: a non-power-of-two dimension is assumed not to repeat. `D_log2` returns
the position of the highest set bit.

**Notes** — the 255 fallback means a non-power-of-two texture's coordinate wraps at 256 rather than at its
real width, which would sample outside it — but the format requires multiples of 16 and every shipped
texture is a power of two in practice. A rebuild should reject non-power-of-two textures explicitly.

## `D_CacheSurface`

**Contract** — takes a surface and a mip level; returns its cached lit form, rebuilding it if absent or
invalid. Resolves the texture through its animation cycle and reads the four light styles' current levels
*before* testing validity, because those are what validity is tested against.

```text
FUNCTION d_cache_surface(surface, miplevel) -> SurfCache
  # Resolve the animated texture and read the current light levels.
  r_drawsurf.texture = the texture, advanced through its animation cycle
  FOR EACH style slot i IN 0..3
    r_drawsurf.lightadj[i] = d_lightstylevalue[surface.styles[i]]

  cache = surface.cachespots[miplevel]
  IF cache EXISTS
     AND cache was NOT built with a dynamic light
     AND no dynamic light touches the surface THIS frame
     AND cache.texture == r_drawsurf.texture
     AND all four of cache.lightadj match
    RETURN cache                                # still valid

  # --- rebuild ---
  surfscale = 1 / (1 SHIFTED LEFT miplevel)
  r_drawsurf.surfmip    = miplevel
  r_drawsurf.surfwidth  = surface.extents[0] SHIFTED RIGHT miplevel
  r_drawsurf.rowbytes   = r_drawsurf.surfwidth
  r_drawsurf.surfheight = surface.extents[1] SHIFTED RIGHT miplevel

  IF cache IS nothing                            # an animated texture reuses
                                                 # the existing allocation
    cache = d_sc_alloc(surfwidth, surfwidth * surfheight)
    surface.cachespots[miplevel] = cache
    cache.owner = &surface.cachespots[miplevel]
    cache.mipscale = surfscale

  cache.dlight = 1 IF a dynamic light touches the surface this frame ELSE 0
  r_drawsurf.surfdat = cache.data
  cache.texture = r_drawsurf.texture
  cache.lightadj = r_drawsurf.lightadj
  r_drawsurf.surf = surface
  r_draw_surface()                               # the CALLBACK UP; see below
  RETURN surface.cachespots[miplevel]
```

**Invariants** — six things here define how lighting reaches the screen.

**The validity test has five conditions and every one is necessary.** The texture must match, or an
animated texture would show a stale frame. All four light levels must match, or a changed light style
would not update. And a surface touched by a dynamic light **this frame**, or built while touched by one,
is always rebuilt — because a dynamic light moves and its contribution cannot be cached.

**A dynamic light therefore rebuilds every surface it touches, every frame.** That is the cost model of
dynamic lighting in this renderer, and it is why a rocket flying down a corridor is measurably expensive.

**A changed animation frame reuses the allocation** rather than reallocating, because the dimensions are
the same. The comment says so explicitly, and it matters: an animated texture would otherwise churn the
cache every two tenths of a second.

**The surface is built by calling *up* into the renderer** ([`d_iface.h`](d_iface.h.md#the-call-up)).
The cache discovers it needs a surface and asks the renderer to produce it, which is the one inversion in
the renderer-rasterizer split and the reason the two halves are not cleanly layered.

**The cached dimensions are the surface's extents shifted by the mip level**, and the extents are already
multiples of 16 ([`model.c`](model.c.md#calcsurfaceextents)) — so at mip level 3 a surface is at least one
texel in each direction and the shift never rounds to zero.

**The owner back-pointer is set to the surface's own cache slot**, which is how eviction clears it. That
pointer is the whole invalidation mechanism.
