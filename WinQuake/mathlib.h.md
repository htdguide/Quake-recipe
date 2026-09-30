# WinQuake/mathlib.h

> Declares the vector and fixed-point vocabulary the whole engine speaks, and the one geometric predicate that is fast enough to be a macro.

**Needs** — nothing; this is a leaf
**Used by** — every module in the engine. Notable direct consumers: [`mathlib.c`](mathlib.c.md) · [`world.c`](world.c.md) · [`r_main.c`](r_main.c.md) · [`sv_phys.c`](sv_phys.c.md) · [`model.c`](model.c.md)
**Tier floor** — none: arithmetic only

## Purpose

Every position, direction, angle triple and plane in the engine is a run of three
single-precision floats, and this file is where that decision is recorded. It
also fixes the three fixed-point formats the rasterizer and the mixer use, and it
supplies the bounding-box-versus-plane test in a form that answers the common
case without a call.

The split between this file and [`mathlib.c`](mathlib.c.md) is not arbitrary:
what lives here is what must be inlined at the call site or must be visible as a
type, and what lives there is what may be a call.

## State

Stateless, apart from two constants every module reads.

```text
CONSTANT vec3_origin : vec3 = (0, 0, 0)
CONSTANT nanmask     : int  = 255 SHIFTED LEFT 23   # the exponent field of a
                                                    # single-precision float,
                                                    # all ones
```

## Types

```text
TYPE vec       = real (32-bit IEEE 754 single)
TYPE vec3      = vec[3]        # ordered x, y, z — or, for an angle triple,
                               # pitch, yaw, roll
TYPE vec5      = vec[5]        # a vec3 plus a two-dimensional texture
                               # coordinate, used only by the sky warp

TYPE fixed4    = int    # 28.4  fixed point
TYPE fixed8    = int    # 24.8  fixed point
TYPE fixed16   = int    # 16.16 fixed point
```

The three fixed-point widths are load-bearing, not stylistic. `fixed4` is the
subpixel precision of the edge rasterizer's x coordinate; `fixed8` is the
intermediate for the eight-bit-fraction reciprocal used in perspective
correction; `fixed16` is the texture coordinate stepped along a span. A rebuild
that widens any of them changes which pixel a polygon edge lands on, which is
visible when comparing a recorded demo frame by frame.

## Angle convention

Three named axis indices are fixed here by the constants `PITCH = 0`,
`YAW = 1`, `ROLL = 2` (declared in [`quakedef.h`](quakedef.h.md) but only
meaningful alongside this file). An angle triple is therefore *not* in x-y-z
order: index 0 rotates the nose down, index 1 turns left, index 2 rolls.
Positive pitch looks **down**. This sign is the single most copied-wrong detail
in the file, and it is baked into the wire protocol, the model data and the game
logic.

## `IS_NAN`

**Contract** — takes a single-precision float; returns true when its exponent
field is all ones, which covers both infinities and both quiet and signalling
not-a-numbers. Reads the float's bit pattern, does no arithmetic, and therefore
never itself raises an exception on a signalling value — which is the reason it
exists rather than the obvious comparison.

**Notes** — used to reject corrupt values arriving from game logic before they
reach the physics code. A rebuild in a language whose float comparison already
handles this safely should use its own predicate.

## `DotProduct`, `VectorSubtract`, `VectorAdd`, `VectorCopy`

**Contract** — the four operations that appear in inner loops, with the obvious
meanings. Each takes three-component operands and, apart from the dot product,
writes into a caller-supplied destination that may alias a source.

**Notes** — these four exist twice: once here for the call sites that cannot
afford a call, and once in [`mathlib.c`](mathlib.c.md) as ordinary operations for
the places that want a function value. Nothing in the recipe depends on the
duplication; a rebuild writes each once.

## `BOX_ON_PLANE_SIDE`

**Contract** — takes a bounding box as a minimum and a maximum corner and a
plane; returns 1 when the box lies wholly on the plane's front side, 2 when
wholly behind, 3 when it straddles. For an axis-aligned plane it decides from a
single coordinate comparison; otherwise it defers to
[`BoxOnPlaneSide`](mathlib.c.md#boxonplaneside).

```text
FUNCTION box_on_plane_side(mins, maxs, plane) -> int
  IF plane.type < 3                      # plane normal is +x, +y or +z
    IF plane.dist <= mins[plane.type]  RETURN 1    # box entirely in front
    IF plane.dist >= maxs[plane.type]  RETURN 2    # box entirely behind
    RETURN 3                                        # box straddles
  RETURN box_on_plane_side_general(mins, maxs, plane)
```

**Invariants** — the result is never 0: a box always has a side, and code
downstream treats 0 as a corruption. The three return values are a bit set, so
"straddles" is exactly "front and back".

**Notes** — the axial fast path is worth a macro because roughly three quarters
of the planes in a compiled map are axis-aligned, and this predicate is the
inner test of the BSP walk, the visibility cull and the collision trace. The
`type` field it keys on is precomputed at map compile time
([`bspfile.h`](bspfile.h.md)) purely so this test can exist; the source's own
comment notes it is trivial to regenerate, which a rebuild that dislikes
redundant on-disk data should do at load.
