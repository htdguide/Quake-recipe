# WinQuake/anorm_dots.h

> A precomputed brightness table: for each of sixteen light directions and each of the 162 model normals, the shading multiplier — so that lighting a model vertex is one table lookup and no arithmetic.

**Needs** — [`anorms.h`](anorms.h.md), from which it is derived
**Used by** — [`r_alias.c`](r_alias.c.md) · [`gl_mesh.c`](gl_mesh.c.md) · [`gl_rmain.c`](gl_rmain.c.md)
**Tier floor** — none

## Purpose

Model lighting in this engine costs **one indexed load per vertex**. This is how: the light direction is
quantized to one of sixteen yaw steps, that selects a row, the vertex's normal index selects a column,
and the value is the brightness multiplier. No dot product, no normalization, no branch.

## State

```text
CONSTANT shadedots : real[16][256]     # [quantized light yaw][normal index]
```

**Invariants** — sixteen rows, so the light direction is quantized to **22.5 degrees of yaw** and the
elevation is ignored entirely. So a model's lighting changes in sixteen discrete steps as it turns, and
a light directly above it looks the same as one at the horizon. That is the approximation, and it is
visible: a rotating item's shading steps rather than sweeping.

Each row has **256 columns although only 162 normals exist**, and the trailing 94 entries are all 1.0.
So an out-of-range normal index reads a neutral value rather than past the table — deliberate padding as
a safety net.

The values range from about 0.70 to 2.00, centred on 1.0. They are **not** clamped cosines: the range
above 1 means a vertex facing the light is brightened above the base level, not merely unshaded. That is
what gives models their slightly overexposed lit side.

```text
FUNCTION build_shadedots() -> real[16][256]
  # The original's generator is not in the tree. The relationship is:
  FOR EACH light yaw step i IN 0..15
    lightdir = the unit vector at yaw (i * 22.5 degrees), elevation 0
    FOR EACH normal index j IN 0..161
      # A shading curve over the cosine, scaled into roughly 0.7 .. 2.0
      shadedots[i][j] = shade_curve(dot(bytedirs[j], lightdir))
    FOR EACH j IN 162..255
      shadedots[i][j] = 1.0
```

**Notes** — the exact curve is not recoverable from the table alone, and the generator is not in the
release. A rebuild that wants the original's look must **copy the table**; a rebuild willing to look
slightly different can compute a clamped cosine scaled into the same range, and the difference is a
modest change in how strongly models are shaded.

The table is 16 by 256 single-precision floats, which is 16 kilobytes of the binary. A rebuild could
halve it by storing bytes.
