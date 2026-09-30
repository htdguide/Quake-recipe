# WinQuake/d_part.c

> Draws one particle: projects it, chooses a square size from its depth, and writes that many depth-tested pixels.

**Needs** — [`d_local.h`](d_local.h.md) · [`d_iface.h`](d_iface.h.md) · [`r_local.h`](r_local.h.md)
**Used by** — [`r_part.c`](r_part.c.md) calls it once per live particle; [`d_parta.s`](d_parta.s.md) is its accelerated twin
**Tier floor** — none

## Purpose

Every explosion, blood spray and rocket trail in the game is a few hundred calls to this function. It is
worth reading for one decision: **a particle is a screen-aligned square of between one and four pixels,
chosen from depth, not a scaled sprite.** That is why particles in this game look like pixels rather than
like puffs, and why they visibly step between sizes as they recede.

## State

Stateless; reads the view basis and the precomputed particle bounds from
[`d_modech.c`](d_modech.c.md).

## `D_DrawParticle`

**Contract** — takes one particle; transforms it into view space, rejects it if nearer than the clip
distance, projects it, rejects it if outside the inset view bounds, chooses a size, and writes a square of
that size, depth-testing and depth-writing each pixel.

```text
FUNCTION d_draw_particle(p)
  local = p.org - r_origin
  transformed = (dot(local, r_pright), dot(local, r_pup), dot(local, r_ppn))
  IF transformed[2] < 8.0  RETURN                   # the particle z clip

  zi = 1 / transformed[2]
  u = round(xcenter + zi * transformed[0])
  v = round(ycenter - zi * transformed[1])          # screen y is inverted
  IF (u, v) outside the INSET particle bounds  RETURN

  pz    = the depth buffer AT (v, u)
  pdest = the view buffer  AT (v, u)
  izi = truncate(zi * 0x8000)

  # The size: the depth reciprocal shifted by a resolution-derived amount,
  # clamped into the resolution-derived range.
  pix = izi SHIFTED RIGHT d_pix_shift
  clamp pix INTO d_pix_min .. d_pix_max

  # Write a pix-by-pix square, its height doubled on a tall-pixel display.
  rows = pix SHIFTED LEFT d_y_aspect_shift
  FOR EACH of `rows` rows
    FOR EACH of `pix` columns i
      IF the stored depth <= izi
        store izi ;  store p.color
    advance both pointers by one row
```

**Invariants** — five things.

**The bounds test uses the *inset* view rectangle**, inset by the maximum particle size
([`d_modech.c`](d_modech.c.md)). So a particle whose centre passes the test cannot write outside the view,
and the inner loops need no clipping at all. That is why the insetting exists.

**The size is the depth reciprocal shifted, not divided.** The shift amount and the clamp bounds are
derived from the resolution so that a particle's apparent size is resolution-independent
([`d_modech.c`](d_modech.c.md)). At the base resolution the range is one to four pixels.

**The height is doubled on a display with tall pixels**, by one shift, so a particle is square on screen
rather than in pixels.

**Every pixel is depth-tested and depth-written**, so particles occlude each other and are occluded by the
world — which is what the world pass's depth writes are for
([`d_local.h`](d_local.h.md#the-depth-buffer)).

**The near clip is 8 world units**, which is much nearer than the general near clip
([`r_local.h`](r_local.h.md)); a particle closer than that would be enormous, so it is dropped.

**Notes** — the source writes the size cases 1 through 4 out longhand and falls back to a loop beyond, which
is pure unrolling. The general case is the contract.

## `D_StartParticles`, `D_EndParticles`

**Contract** — both do nothing. They exist because a rasterizer that batches particles would need them
([`d_iface.h`](d_iface.h.md)).
