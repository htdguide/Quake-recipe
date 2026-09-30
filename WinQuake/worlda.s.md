# WinQuake/worlda.s

> An x86 implementation of the collision tree's point query, selected instead of the portable one when the assembly build is enabled.

**Needs** — [`asm_i386.h`](asm_i386.h.md) · [`quakeasm.h`](quakeasm.h.md) · [`d_ifacea.h`](d_ifacea.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops)
**Used by** — [`world.c`](world.c.md), which omits its own version of the same routine when this one is compiled
**Tier floor** — T0 as written; the contract it implements is tier-free

## Purpose

One routine, hand written, with the same contract as the portable version in
[`world.c`](world.c.md#sv_hullpointcontents). It exists because the point query is called
several times per collision sweep — once at every plane crossing — and several thousand times
per server frame.

A rebuild implements the contract and ignores this file. What is worth carrying out of it is
**why** the contract can be implemented this tightly, because that shapes how a rebuild
should lay out its data.

## State

Stateless, apart from one word of scratch space used to move a value between the integer and
floating-point units.

## `SV_HullPointContents`

**Contract** — identical to [`world.c`](world.c.md#sv_hullpointcontents): descend a
collision tree from a given node, comparing a point against each node's plane, until a
negative value is reached, and return it. A node index at or above zero but outside the
tree's range is **not** checked here — see the note.

```text
FUNCTION sv_hull_point_contents(hull, num, p) -> int
  IF num < 0  RETURN num                      # already a contents value
  clipnodes = hull.clipnodes ;  planes = hull.planes
  LOOP
    node  = clipnodes[num]                    # index scaled by the record size
    plane = planes[node.planenum]             # likewise
    d = (p[plane.type] - plane.dist)          IF plane.type < 3
        (dot(plane.normal, p) - plane.dist)   OTHERWISE
    num = node.children[1] IF d < 0 ELSE node.children[0]
    IF num < 0  RETURN num
```

**Invariants** — the sign test on the distance is performed by examining the **sign bit of
the floating-point result** rather than by a comparison, so a distance of negative zero takes
the *back* child while the portable version's `d < 0` test takes the front. Negative zero
arises when the point lies exactly on a plane whose distance is zero — the world origin — so
the two implementations can disagree there. Nothing in a shipped map puts collision geometry
through the origin, and the disagreement has never been observed; a rebuild should implement
the portable form's comparison.

The routine relies on three layout facts, each called out by a warning comment in the source:
the collision node record's size, the plane record's size, and the plane's field order. Each
index is scaled by a constant, so **changing any of those sizes silently breaks this file**.
The header it includes exists to keep the constants in one place, and it is a compile-time
copy of layout decisions made in C.

## Why there is no range check

The portable version raises a fatal error on an out-of-range node index. This one does not:
it tests only the sign. The two therefore differ on a corrupt map, where the portable build
aborts with a diagnostic and the assembly build reads memory outside the tree.

A rebuild should validate node indices **once at load time**, which removes the check from
the hot path entirely and makes the range test unnecessary in either implementation. That is
the right resolution of the difference, and it is what
[`bspfile.h`](bspfile.h.md) already implies: the index fields are 16-bit and the counts are
known when the map is read.

## What a rebuild should take from this

The routine is fast because the data is laid out for it: nodes and planes are each a flat
array of fixed-size records, a child is an index rather than a pointer, and the plane's axis
classification is precomputed so the common case is one subtraction. Those four properties
are the load-bearing content of this file, and they belong to
[`bspfile.h`](bspfile.h.md) rather than to any implementation.

The 1996 machine's cost model — where a floating-point compare-and-branch was expensive and
a sign-bit test was not — no longer holds. A rebuild should write the portable version and
measure before doing anything else.
