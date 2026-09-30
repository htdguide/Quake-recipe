# WinQuake/chase.c

> The third-person camera: places the view behind the player and pulls it forward until it is clear of geometry.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`render.h`](render.h.md) · [`world.h`](world.h.md) · [`mathlib.h`](mathlib.h.md) · [`cvar.h`](cvar.h.md)
**Used by** — [`host.c`](host.c.md) initializes it; [`view.c`](view.c.md) applies it; [`cl_main.c`](cl_main.c.md) checks whether it is active before skipping the player's own entity
**Tier floor** — none

## Purpose

Ninety lines, and the only decision in it worth recording is how the camera avoids ending up inside a wall: it
traces from the player outward to the desired position and stops short of whatever it hits.

## State

```text
VARIABLE chase_back, chase_up, chase_right, chase_active : Cvar
```

## `Chase_Init`, `Chase_Reset`

**Contract** — register the four variables; and reset the camera's state.

## `TraceLine`

**Contract** — traces a line through the world only, ignoring entities, and returns the impact point — the
endpoint itself when nothing was hit.

**Invariants** — **entities are ignored**, so the camera passes through monsters and items but not through
geometry. That is deliberate: a camera that collided with entities would jump about in a crowd.

## `Chase_Update`

**Contract** — computes the desired camera position from the player's position and view direction offset by the
three configured distances, traces from the player to it, and places the view at the impact point pulled a few
units back toward the player. Points the view at the player.

```text
FUNCTION chase_update()
  forward, right, up = the basis OF the player's view angles
  # The desired position: behind, above and beside the player.
  dest = the player's origin
         - forward * chase_back
         + up      * chase_up
         + right   * chase_right
  # Pull it in until it is clear of geometry.
  impact = trace_line(the player's origin, dest)
  the view position = impact, moved 8 units BACK toward the player
  the view angles = the direction FROM the view position TO the player
```

**Invariants** — the eight-unit pull-back is what keeps the camera's near plane outside the surface it stopped
against; without it the wall clips through the view.

The view *angles* are re-derived to look at the player rather than kept from the player's own, which is what
makes the camera track rather than merely follow.

**Notes** — the player's own entity is drawn in this mode, which is why
[`cl_main.c`](cl_main.c.md#cl_relinkentities) checks the active flag before skipping it. That one check is the
whole of the mode's integration with the renderer.
