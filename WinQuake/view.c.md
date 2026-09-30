# WinQuake/view.c

> The camera: the view bob, the velocity lean, the automatic pitch centring, the damage kick, the four-layer screen tint, and the weapon's lagged aim — every piece of feel that is not movement.

**Needs** — [`client.h`](client.h.md) · [`view.h`](view.h.md) · [`render.h`](render.h.md) · [`protocol.h`](protocol.h.md) · [`vid.h`](vid.h.md) · [`screen.h`](screen.h.md) · [`bspfile.h`](bspfile.h.md) · [`cvar.h`](cvar.h.md) · [`cmd.h`](cmd.h.md) · [`mathlib.h`](mathlib.h.md)
**Used by** — [`host.c`](host.c.md) initializes it; [`screen.c`](screen.c.md) calls the render; [`sv_user.c`](sv_user.c.md) shares the lean calculation; [`cl_parse.c`](cl_parse.c.md) feeds it damage
**Tier floor** — none

## Purpose

This file is why the game feels the way it does. Nothing in it is required for correctness; every routine adds a
small motion or tint that the player reads as physicality. The numbers are all tunable and all default to values
chosen by feel.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## `V_CalcRoll`

**Contract** — takes an angle triple and a velocity; returns the view lean, proportional to the sideways
component of the velocity up to a threshold and constant beyond it. Shared with the server
([`sv_user.c`](sv_user.c.md#sv_clientthink)) so that a player's own view and their model's lean agree.

```text
FUNCTION v_calc_roll(angles, velocity) -> real
  right = the right vector OF angles
  side = dot(velocity, right)
  sign = the sign of side ;  side = |side|
  IF side < cl_rollspeed  side = side * cl_rollangle / cl_rollspeed
  ELSE                    side = cl_rollangle
  RETURN side * sign
```

**Invariants** — the lean saturates at a configured angle once the sideways speed passes a threshold, so
strafing at full speed leans a fixed amount rather than growing without bound. **This is the only function
shared between the client's rendering and the server's simulation**, and it must stay shared or the two disagree.

## `V_CalcBob`

**Contract** — returns the vertical view bob: a sine of a cycle position, scaled by the player's horizontal
speed, clamped asymmetrically.

```text
FUNCTION v_calc_bob() -> real
  # Position within the bob cycle, in 0..1.
  cycle = (cl.time MODULO cl_bobcycle) / cl_bobcycle
  # Warp the cycle so the up-stroke and down-stroke can differ in duration.
  IF cycle < cl_bobup  cycle = pi * cycle / cl_bobup
  ELSE                 cycle = pi + pi*(cycle - cl_bobup)/(1 - cl_bobup)
  # Scale by HORIZONTAL speed only — counting z would make jumping bob.
  bob = sqrt(vx*vx + vy*vy) * cl_bob
  bob = bob*0.3 + bob*0.7*sin(cycle)         # 30% constant, 70% oscillating
  clamp bob INTO -7 .. 4
  RETURN bob
```

**Invariants** — four things.

**The cycle is warped so the up-stroke and the down-stroke have different durations**, controlled by a tunable.
That asymmetry is what makes the bob read as footfalls rather than as a sine wave.

**The vertical velocity is excluded**, and the comment says why: counting it would make jumping bob.

**The bob is 30 percent constant and 70 percent oscillating**, so a moving player's view sits slightly higher
than a stationary one's as well as oscillating.

**The clamp is asymmetric — 4 up, 7 down** — so a very fast player dips more than they rise.

## `V_DriftPitch`

**Contract** — moves the view's pitch toward the server's suggested ideal pitch, accelerating, but only while the
player is on the ground and has been moving forward for long enough. Cancelled by any manual look.

```text
FUNCTION v_drift_pitch()
  IF noclipping OR not on the ground OR playing back a demo
    cancel the drift ;  RETURN
  IF the drift is not armed
    # Arm it only after sustained forward movement, so a small mouse nudge
    # does not start it.
    IF |cl.cmd.forwardmove| < cl_forwardspeed  cl.driftmove = 0
    ELSE                                       cl.driftmove += host_frametime
    IF cl.driftmove > v_centermove  arm the drift
    RETURN
  delta = cl.idealpitch - cl.viewangles[PITCH]
  IF delta == 0  cl.pitchvel = 0 ;  RETURN
  move = host_frametime * cl.pitchvel
  cl.pitchvel = cl.pitchvel + host_frametime * v_centerspeed   # ACCELERATES
  move toward the ideal by `move`, stopping exactly at it and zeroing the
  velocity when it would overshoot
```

**Invariants** — three things.

**The drift accelerates rather than moving at a constant rate**, so it begins imperceptibly and completes
quickly. A constant rate reads as the view being dragged.

**It arms only after sustained forward movement**, which is the source's stated intent: a small mouse motion
should not trigger it. That is why it needs both a threshold and a duration.

**It is suspended while airborne**, because the ideal pitch is derived from the floor ahead
([`sv_user.c`](sv_user.c.md#sv_setidealpitch)) and is meaningless in the air.

## `V_ParseDamage`

**Contract** — decodes the damage message: armour absorbed, health lost, and the position of whatever inflicted
it. Adds a red tint proportional to the total, kicks the view angles away from the source, and flashes the
status bar's face.

**Invariants** — the kick's *direction* comes from the inflictor's position relative to the player, so being shot
from the left kicks the view right. That directionality is the whole point of the message carrying a position.

The tint's strength is proportional to the damage and its colour is biased by the armour-to-health ratio, so an
armoured hit tints differently from a bare one.

## `V_SetContentsColor`, `V_CalcPowerupCshift`, `V_cshift_f`, `V_BonusFlash_f`

**Contract** — set the contents tint layer from the contents value the view is inside; set the powerup layer from
the player's items; override the contents layer from the console; and set the bonus layer for an item pickup.

**Invariants** — the four layers are independent and composited
([`client.h`](client.h.md)), which is why taking damage underwater while holding a powerup shows all three.

## `V_CalcBlend`

**Contract** — composites the four tint layers into one colour and one strength, in order, each blended over the
accumulated result by its own percentage. Clamps the strength.

```text
FUNCTION v_calc_blend()
  r = g = b = a = 0
  FOR EACH of the four layers j
    a2 = the layer's percentage, scaled, as a fraction
    IF a2 == 0  CONTINUE
    a = a + a2*(1 - a)                 # accumulate coverage
    a2 = a2 / a                        # the layer's share OF the total
    r = r*(1 - a2) + the layer's colour * a2      # and the same for g and b
  v_blend = (r, g, b, a) WITH the colours as fractions and a clamped to 1
```

**Invariants** — this is standard **over** compositing done in one pass, with the running coverage used to
normalize each layer's contribution. The order is fixed, so a damage flash sits over a powerup tint rather than
under it.

## `V_UpdatePalette`

**Contract** — recomputes the tint layers and, when the result has changed, rebuilds the whole 256-entry palette
by blending each entry toward the tint colour by the tint strength, passing each channel through the gamma
table, and installs it.

**Invariants** — **the tint is applied by rebuilding the palette**, so it affects *everything* including the
status bar and the console. That is why a damage flash in this game tints the interface as well as the world, and
it is the defining difference from the hardware renderer, which blends a quad over the view only.

The rebuild is skipped when neither the tint nor the gamma changed, because it is 768 multiplications and a
palette upload.

A second implementation of the same function exists for the hardware build and does nothing but record the blend
for the renderer to use.

## `BuildGammaTable`, `V_CheckGamma`

**Contract** — build the 256-entry gamma table from an exponent, and rebuild it when the setting changed,
forcing a palette update.

## `CalcGunAngle`, `angledelta`

**Contract** — computes the weapon model's angles as a **lagged** version of the view angles, easing toward them
at a rate proportional to the difference and clamped.

**Invariants** — the weapon lags the view, which is what makes turning feel weighty. The lag is per axis and
bounded, so the weapon never trails far enough to leave the screen.

## `V_BoundOffsets`

**Contract** — clamps the view position to stay within a small box around the player's origin, so that the
accumulated bob, lean and configured offsets cannot put the camera inside a wall.

## `V_AddIdle`, `V_CalcViewRoll`

**Contract** — add a slow sinusoidal drift to all three view angles, scaled by a tunable that defaults to zero;
and apply the velocity lean plus, when the player is dead, a fixed death roll.

## `V_CalcRefdef`

**Contract** — composes the view for one frame: drifts the pitch, points the player's model along the view,
places the view at the player's eye height plus the bob, nudges it off any axis plane, applies the lean, the idle
drift and the configured offsets, bounds the result, and places the weapon model with its own lagged angles and a
fraction of the bob. Smooths a step up.

```text
FUNCTION v_calc_refdef()
  v_drift_pitch()
  ent  = the player's own entity ;  view = the weapon model
  # The player MODEL faces the view direction, with pitch NEGATED.
  ent.angles[YAW]   =  cl.viewangles[YAW]
  ent.angles[PITCH] = -cl.viewangles[PITCH]
  bob = v_calc_bob()

  r_refdef.vieworg = ent.origin, raised BY cl.viewheight + bob
  # Never sit exactly on a node plane: a water surface can vanish when the eye
  # is exactly on it. The protocol quantizes to 1/8, so add 1/32 on each axis.
  r_refdef.vieworg = r_refdef.vieworg + (1/32, 1/32, 1/32)

  r_refdef.viewangles = cl.viewangles
  v_calc_view_roll() ;  v_add_idle()
  apply the three configured offsets ALONG the player's own basis
  v_bound_offsets()

  view.angles = cl.viewangles ;  calc_gun_angle()
  view.origin = ent.origin, raised by cl.viewheight
  view.origin = view.origin + forward * bob * 0.4, and raised by bob
  # Smooth a step up: the view rises toward the new height rather than jumping.
  IF the player is on the ground AND rose since last frame
    accumulate the difference into cl.crouch and subtract it from the view,
    decaying it over a fraction of a second
```

**Invariants** — five things.

**The one-thirty-second nudge on every axis** has its reason stated in the source: the eye exactly on a node
plane can make a water surface disappear, because the tree walk puts the viewpoint on one side and the surface
is on the other. The protocol quantizes coordinates to eighths
([`common.c`](common.c.md#msg_writecoord)), so a thirty-second is guaranteed to break the tie without being
visible.

**The player model's pitch is negated relative to the view's**, because entity pitches run the other way
([`mathlib.c`](mathlib.c.md#anglevectors)) — the same negation
[`sv_user.c`](sv_user.c.md#sv_clientthink) applies on the server.

**The weapon gets 40 percent of the bob forward and 100 percent vertically**, so it bobs less than the view
horizontally. Two disabled lines show the sideways and vertical variants that were tried and rejected.

**The step-up smoothing is purely local** ([`client.h`](client.h.md)): the server teleports the player up a
step instantly and the client eases the *camera* up over a fraction of a second. Without it, climbing stairs
jolts the view once per step.

**The configured view offsets are applied along the player's own basis**, not the world's, so they follow the
player's facing.

## `V_CalcIntermissionRefdef`

**Contract** — composes the view for the end-of-level screen: a fixed camera at the position the server set, with
no bob, lean or drift.

## `V_RenderView`

**Contract** — composes the view — the intermission variant when at the end of a level, the chase camera's when
active, the normal one otherwise — pushes the dynamic lights, and calls the renderer. Does nothing before the
handshake completes.

## `V_Init`

**Contract** — registers the twenty-odd feel tunables and the two console commands.
