# WinQuake/gl_test.c

> A scratch file: a handful of self-moving puffs drawn as blended quads, used to try the blending path.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`glquake.h`](glquake.h.md)
**Used by** — nothing in the shipped build
**Tier floor** — none

## Purpose

Developer scratch, left in the tree and not called. It spawns a few objects with a position, a velocity and a lifetime,
advances them each frame, and draws each as a blended quad facing the viewer.

## State

```text
RECORD Puff
  origin, velocity : vector
  die : time
VARIABLE a small fixed array of them
```

## The operations

**Contract** — initialize the array; spawn one at the viewer's position with a random velocity; advance and draw them
all, removing those whose time has passed.

**Notes** — skip it. Recorded because the recipe mirrors the tree. The one thing it documents is the minimal shape of a
blended billboard in this renderer, which [`gl_rmain.c`](gl_rmain.c.md#r_getspriteframe-r_drawspritemodel) does properly.
