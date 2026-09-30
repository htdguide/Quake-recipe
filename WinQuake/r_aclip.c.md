# WinQuake/r_aclip.c

> Clips a model triangle against the near plane and the four screen edges, generating the vertices where it crosses.

**Needs** — [`r_local.h`](r_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`model.h`](model.h.md) · [Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops) ([`r_aclipa.s`](r_aclipa.s.md))
**Used by** — [`r_alias.c`](r_alias.c.md), for a model the frustum cut
**Tier floor** — none

## Purpose

Five clip routines and a triangle clipper. The five differ in a way worth naming: the **depth** clip
interpolates in world space before projection, while the four **screen-edge** clips interpolate the already
projected values. That asymmetry is necessary — a vertex behind the near plane has no meaningful projection —
and it is why depth clipping is separate.

## State

Stateless.

## `R_Alias_clip_z`

**Contract** — takes two vertices straddling the near plane; produces the vertex on the plane by interpolating
their **view-space** positions and texture coordinates, then projecting the result.

**Invariants** — the near plane is 5 units in the model's scaled space
([`r_local.h`](r_local.h.md)). Interpolating before projection is required because one endpoint's projection
would be behind the camera.

## `R_Alias_clip_left`, `_right`, `_top`, `_bottom`

**Contract** — each takes two vertices straddling one screen boundary; produces the vertex on that boundary by
interpolating every already-projected value — screen position, texture coordinates, light and depth reciprocal
— and **assigning the boundary coordinate exactly**.

**Invariants** — the exact assignment, rather than an interpolated value, is what stops a clipped vertex landing
a fraction outside the view and letting the rasterizer write past the buffer. Four separate routines rather
than one parameterized, because each assigns a different coordinate
([`r_aclipa.s`](r_aclipa.s.md) notes the same).

## `R_AliasClipTriangle`

**Contract** — takes a triangle; clips it against the near plane and then each screen edge in turn, and emits
the resulting polygon as a fan of triangles to the rasterizer.

```text
FUNCTION r_alias_clip_triangle(ptri)
  vertices = the triangle's three final vertices
  FOR EACH clip boundary IN { depth, left, right, top, bottom }
    IF no vertex is outside this boundary  CONTINUE
    IF every vertex is outside             RETURN        # reject
    # Walk the polygon, keeping inside vertices and generating one where an
    # edge crosses.
    output = empty
    FOR EACH edge (a, b) OF the polygon
      IF a is inside  append a
      IF a and b are on opposite sides
        append the clip routine's result FOR this boundary
    vertices = output
  # Emit the result as a fan.
  FOR EACH i FROM 1 TO count-2
    hand the rasterizer the triangle (vertices[0], vertices[i], vertices[i+1])
```

**Invariants** — clipping in sequence against five boundaries can grow a triangle to at most eight vertices,
which is why the working arrays are sized accordingly. A polygon reduced below three vertices is rejected.

**Notes** — the source asks whether the fan could be emitted all at once. A rebuild using a triangle fan
primitive can.
