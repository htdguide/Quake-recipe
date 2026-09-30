# WinQuake/mathlib.c

> The engine's arithmetic floor: vector algebra, the angle-triple-to-basis conversion every camera and projectile depends on, and the exact rounding rules the rasterizer is built on.

**Needs** — [`mathlib.h`](mathlib.h.md) · [`quakedef.h`](quakedef.h.md) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services) (for the fatal-error exit) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — everything. The named callers of the less obvious entry points: [`r_alias.c`](r_alias.c.md) and [`gl_rmain.c`](gl_rmain.c.md) for the transform concatenation, [`d_surf.c`](d_surf.c.md) and [`r_surf.c`](r_surf.c.md) for the reciprocal, [`r_sky.c`](r_sky.c.md) for the floor-based division, [`snd_dma.c`](snd_dma.c.md) for the greatest common divisor
**Tier floor** — none, but see the rounding note: a rebuild needs a language whose float-to-int conversion it can pin down

## Purpose

Two kinds of thing live here, and telling them apart matters. Most of the file is
plain vector algebra whose only interesting property is that it exists — any
rebuild writes it without reading this page. The rest is a small set of
operations whose *exact* results are observable downstream: the angle-to-basis
conversion, the modulo that wraps an angle, the floor-based division, and the
8.24-to-16.16 reciprocal. Those four are specified precisely here because getting
them approximately right produces an engine that looks correct and desynchronizes
from a recorded demo within seconds.

Several routines in this file have an assembly twin selected at compile time (see
the seam). The contracts below are the specification both implementations meet.

## State

Stateless. The two constants it defines are documented in
[`mathlib.h`](mathlib.h.md).

## `AngleVectors`

**Contract** — takes an angle triple in degrees and produces three orthonormal
basis vectors: forward, right and up. Writes all three; reads nothing else. This
is the conversion from "where the player is looking" to "which way is which",
and every camera, every hitscan trace, every thrown projectile and every sound
spatialization goes through it.

**Invariants** — the three outputs are unit length and mutually perpendicular to
float precision. Forward is the direction of gaze. **Right is the player's right
hand**, and the composition is a left-handed one: with all angles zero, forward
is +x, right is −y, up is +z. Positive pitch pushes forward's z *negative* —
i.e. positive pitch looks down.

```text
FUNCTION angle_vectors(angles) -> (forward, right, up)
  sy, cy = sin, cos OF radians(angles[YAW])
  sp, cp = sin, cos OF radians(angles[PITCH])
  sr, cr = sin, cos OF radians(angles[ROLL])

  forward = ( cp*cy,
              cp*sy,
             -sp )
  right   = ( -sr*sp*cy + cr*sy,
              -sr*sp*sy - cr*cy,
              -sr*cp )
  up      = (  cr*sp*cy + sr*sy,
               cr*sp*sy - sr*cy,
               cr*cp )
```

**Notes** — the source writes the right and up expressions with explicit
multiplications by −1 and double negations, which is how the signs above were
derived; the forms given are algebraically identical. The sign conventions here
are not a choice a rebuild gets to make: they are fixed by the map data, the
model data and the protocol, all of which predate the code.

## `anglemod`

**Contract** — takes an angle in degrees, returns it reduced to the half-open
range [0, 360). Not the obvious floating-point modulo: it quantizes to the same
sixteen-bit grid the network protocol uses, so that an angle that has made a
round trip over the wire and an angle that has not agree exactly.

```text
FUNCTION anglemod(a) -> real
  # 65536 steps over 360 degrees; truncation toward zero, then mask to 16 bits
  RETURN (360 / 65536) * (truncate_to_int(a * (65536 / 360)) BITAND 65535)
```

**Invariants** — the result is always an exact multiple of 360/65536. For a
negative input the truncation-then-mask combination wraps correctly, which is why
the masking replaces a conditional; the commented-out branch-based version in the
source does not produce the same value and is not the contract.

**Notes** — the quantization is the point. Game logic compares an entity's
current yaw against its target yaw for equality-within-a-step, and if those two
values came from different precisions the comparison never settles and monsters
spin in place. A rebuild that "improves" this to a true float modulo introduces
exactly that bug.

## `BoxOnPlaneSide`

**Contract** — takes a bounding box and a non-axial plane; returns the same
1/2/3 side code as [`BOX_ON_PLANE_SIDE`](mathlib.h.md#box_on_plane_side). Reached
only when the plane is not axis-aligned, because the macro handles that case
itself. Fatal error on a plane whose precomputed sign bits are out of range.

**Invariants** — never returns 0.

```text
FUNCTION box_on_plane_side_general(mins, maxs, plane) -> int
  # plane.signbits has bit i set when plane.normal[i] is negative.
  # Pick, per axis, the box corner that maximizes the dot product for the
  # near distance and the one that minimizes it for the far distance.
  FOR EACH axis i IN 0..2
    IF bit i OF plane.signbits IS SET
      near[i] = mins[i] ;  far[i] = maxs[i]
    ELSE
      near[i] = maxs[i] ;  far[i] = mins[i]

  dist_near = dot(plane.normal, near)
  dist_far  = dot(plane.normal, far)

  sides = 0
  IF dist_near >= plane.dist  sides = sides BITOR 1
  IF dist_far  <  plane.dist  sides = sides BITOR 2
  RETURN sides
```

**Notes** — the source does not contain the loop above. It contains an
eight-way switch on the sign bits with the corner selection written out
literally, because in 1996 the branch was cheaper than three conditionals in a
loop and this is one of the hottest predicates in the engine. The loop and the
switch compute the same thing; a rebuild should write the loop and let the
compiler decide. The `signbits` field itself is precomputed when the map loads
([`model.c`](model.c.md)) for the same reason.

The fatal-error arm on an out-of-range sign-bit value is factored into its own
tiny function purely so the assembly implementation has something to call; that
factoring carries no meaning.

## `VectorCompare`

**Contract** — takes two three-vectors, returns true when all three components
are bit-for-bit equal under float comparison. Exact equality, deliberately: it is
used to notice that an entity has not moved at all, and a tolerance there would
suppress updates the protocol needs to send.

## `VectorMA`

**Contract** — multiply-accumulate: takes a base vector, a scalar and a second
vector, writes base + scalar × second into a destination that may alias either
input.

## `Length`, `VectorNormalize`

**Contract** — `Length` returns the Euclidean norm. `VectorNormalize` scales the
vector in place to unit length and returns the length it had; a zero-length
vector is left untouched and zero is returned, rather than producing a
not-a-number.

**Notes** — the zero guard is load-bearing. Normalizing a zero vector happens
routinely — a projectile spawned with no velocity, two coincident entities
looking at each other — and the guard is what keeps that from poisoning
downstream arithmetic with not-a-numbers that [`IS_NAN`](mathlib.h.md#is_nan)
then has to catch.

## `CrossProduct`, `VectorInverse`, `VectorScale`, `_DotProduct`, `_VectorAdd`, `_VectorSubtract`, `_VectorCopy`

**Contract** — the standard operations, each writing into a caller-supplied
destination. The underscore-prefixed four duplicate the macros in
[`mathlib.h`](mathlib.h.md) as callable functions.

**Notes** — the cross product follows the right-hand rule, which sits oddly
beside the left-handed basis produced by `AngleVectors`; both are correct as
written and the engine is consistent about which it uses where. A rebuild that
"fixes" one to match the other breaks the other's callers.

## `Q_log2`

**Contract** — takes a positive integer, returns the position of its highest set
bit, i.e. the floor of the base-two logarithm. Zero returns zero.

**Notes** — used only to turn a power-of-two texture dimension into a shift
count. A rebuild with a count-leading-zeros primitive should use it.

## `R_ConcatRotations`

**Contract** — takes two three-by-three matrices, writes their product into a
third. The destination must not alias either source.

## `R_ConcatTransforms`

**Contract** — takes two three-by-four matrices, each understood as a rotation
in columns 0–2 and a translation in column 3, and writes the composition into a
third. The destination must not alias either source.

```text
FUNCTION concat_transforms(a, b) -> out
  FOR EACH row r IN 0..2
    FOR EACH col c IN 0..2
      out[r][c] = a[r][0]*b[0][c] + a[r][1]*b[1][c] + a[r][2]*b[2][c]
    out[r][3] = a[r][0]*b[0][3] + a[r][1]*b[1][3] + a[r][2]*b[2][3] + a[r][3]
```

**Notes** — the translation column picks up the `+ a[r][3]` term, which is the
whole difference from a pure rotation product. This is the one matrix operation
the animated-model renderer performs per entity per frame; the three-by-four
shape rather than four-by-four is not an optimization detail to preserve, it is
just the shape the callers pass.

## `ProjectPointOnPlane`, `PerpendicularVector`, `RotatePointAroundVector`

**Contract** — `ProjectPointOnPlane` drops a point onto the plane through the
origin with the given normal; the normal need not be unit length. 
`PerpendicularVector` produces some unit vector perpendicular to a given unit
vector, choosing the axis in which the input is smallest so the result is well
conditioned. `RotatePointAroundVector` rotates a point by an angle in degrees
about an arbitrary axis through the origin.

```text
FUNCTION perpendicular_vector(src) -> dst    # src assumed unit length
  pos = index OF THE SMALLEST |src[i]|       # best-conditioned axis
  basis = the unit vector along axis pos
  dst = project_point_on_plane(basis, src)
  normalize(dst)

FUNCTION rotate_point_around_vector(axis, point, degrees) -> dst
  # Build an orthonormal frame whose third column is the rotation axis,
  # rotate about that frame's z, then come back out of the frame.
  vr  = perpendicular_vector(axis)
  vup = cross(vr, axis)
  m   = the matrix with columns (vr, vup, axis)
  im  = transpose(m)                          # m is orthonormal, so this
                                              # is its inverse
  zrot = rotation about z by degrees
  rot = m * zrot * im
  dst = rot * point
```

**Notes** — these three are the only routines in the file with no caller in this
build. They arrived with the mission-pack code, whose rotating-brush entities
needed them, and the mission-pack path is compiled out. A rebuild may omit them
and lose nothing; a rebuild that intends to support rotating map geometry needs
exactly them. The source disables compiler optimization around
`RotatePointAroundVector` on one toolchain, which is a workaround for a 1997
compiler bug and carries no meaning.

## `FloorDivMod`

**Contract** — takes a numerator and a positive denominator, both expected to
have no fractional part, and produces a quotient and remainder using *floor*
semantics rather than the truncation-toward-zero that the language's own division
gives. The quotient must fit in a 32-bit integer. A non-positive denominator is a
fatal error.

```text
FUNCTION floor_div_mod(numer, denom) -> (quotient, remainder)
  IF numer >= 0
    q = floor(numer / denom)
    r = floor(numer - q*denom)
  ELSE
    # work in positives, then correct so that the remainder is non-negative
    x = floor(-numer / denom)
    q = -x
    r = floor(-numer - x*denom)
    IF r != 0
      q = q - 1
      r = denom - r
  RETURN (q, r)
```

**Invariants** — the remainder is always in [0, denom), including for negative
numerators. That is the whole reason the routine exists: it tiles a texture
across a surface whose coordinates go negative, and truncating division would
produce a one-texel seam through the origin.

**Notes** — a rebuild in a language whose modulo already floors (rather than
truncating) can call it directly and delete this.

## `GreatestCommonDivisor`

**Contract** — Euclid's algorithm on two non-negative integers, recursive,
returning the other argument when one is zero.

**Notes** — used once, to reduce a sound's sample rate against the output rate
into a resampling step. Nothing depends on the recursion.

## `Invert24To16`

**Contract** — takes an 8.24 fixed-point value and returns its reciprocal as a
16.16 fixed-point value, rounded to nearest. An input below 256 — that is, a
value whose reciprocal would overflow — returns the all-ones pattern rather than
saturating arithmetically.

```text
FUNCTION invert_24_to_16(val) -> fixed16
  IF val < 256  RETURN 0xFFFFFFFF          # clamp: reciprocal too large
  RETURN truncate_to_int( (65536.0 * 16777216.0 / val) + 0.5 )
```

**Invariants** — the `+ 0.5` before truncation is round-half-up, and it is
observable: this reciprocal is the perspective divide for one span of the
software rasterizer, and a different rounding shifts texture coordinates by a
texel at grazing angles. A rebuild must round the same way, and must perform the
division in double precision — single precision does not have the bits.

**Notes** — the file's own comment marks this as belonging in the
platform-specific arithmetic module ([`nonintel.c`](nonintel.c.md)), which is
where its assembly twin lives. It is here because the C fallback had to live
somewhere.
