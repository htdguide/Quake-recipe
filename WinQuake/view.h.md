# WinQuake/view.h

> The camera: gamma, the screen tint's final blend, and the one function shared between the client's rendering and the server's simulation.

**Needs** — [`mathlib.h`](mathlib.h.md) · [`cvar.h`](cvar.h.md)
**Used by** — [`host.c`](host.c.md) · [`screen.c`](screen.c.md) · [`cl_parse.c`](cl_parse.c.md) · [`sv_user.c`](sv_user.c.md) · implemented by [`view.c`](view.c.md)
**Tier floor** — none

## Purpose

Small, and one declaration in it is structurally interesting: `V_CalcRoll` is called by both the
**client's** view code and the **server's** movement code
([`sv_user.c`](sv_user.c.md#sv_clientthink)). It is the only function in the engine used by both halves
of the simulation, and it exists so that the lean a player sees on their own screen and the lean other
players see on their model are computed identically.

## State

```text
VARIABLE v_gamma    : Cvar
VARIABLE gammatable : byte[256]      # every palette entry passes through this
VARIABLE ramps      : byte[3][256]   # hardware gamma ramps, per channel
VARIABLE v_blend    : real[4]        # the composited screen tint: colour and
                                     # strength
VARIABLE lcd_x      : Cvar           # stereo separation; see the note
```

**Invariants** — gamma is applied by **rebuilding the palette**, not by a hardware ramp, in the software
renderer — so changing it forces a palette upload and, on some backends, a mode reset. The per-channel
ramps exist for backends that can set hardware gamma.

The blend is the four colour-shift layers ([`client.h`](client.h.md)) composited into one colour and
strength, which the palette rebuild then applies to all 256 entries. That is why taking damage in this
game tints *everything* including the interface.

## Entry points

**Contract** — `V_Init` registers the view's variables and commands. `V_RenderView` sets up the view
description and calls the renderer. `V_UpdatePalette` recomposites the tint layers and rebuilds the
palette. `V_CalcRoll` returns the view lean for a given angle triple and velocity.

**Invariants** — the lean is computed from **how much of the velocity is sideways**, scaled and clamped,
so strafing leans the view and running forward does not. Both ends must agree, which is why the function
is shared.

**Notes** — the stereo separation variable drives a left-eye/right-eye rendering mode for a
head-mounted display of the period. It renders the frame twice with offset viewpoints. A rebuild can drop
it, or can keep it as the cheapest possible stereo implementation.
