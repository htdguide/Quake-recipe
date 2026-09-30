# QW/client/pmove.c

> The player movement model, extracted so client and server run it identically: acceleration under friction, a slide move against planes, a stair step chosen by whichever went further, water handling by immersion depth, and the air-acceleration clamp that produced a movement technique nobody designed.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`pmove.h`](pmove.h.md) · [`pmovetst.c`](pmovetst.c.md) · [`mathlib.h`](mathlib.h.md)
**Used by** — [`sv_user.c`](../server/sv_user.c.md) runs it authoritatively; [`cl_pred.c`](cl_pred.c.md) runs it speculatively
**Tier floor** — none

## Purpose

The third of QuakeWorld's three defining ideas. In the original engine, player movement lives inside the server
([`sv_user.c`](../../WinQuake/sv_user.c.md)), interleaved with game logic and entity access, and the client cannot run it.
Over a modem that means every step the player takes appears a round trip late, which is unplayable.

This file is that movement model **lifted out into a pure function** over the interface in [`pmove.h`](pmove.h.md). The client
can then run it on its own commands immediately and correct itself when the server's answer arrives
([`cl_pred.c`](cl_pred.c.md)). Movement becomes instant while remaining server-authoritative.

The extraction is the achievement; the physics below is largely the original's, and where it differs the differences are
called out.

## State

```text
VARIABLE pmove : PlayerMove            # the single shared record
VARIABLE movevars : MoveVars           # the tunables, from the server
VARIABLE onground : int                # -1, or the touched entity's tag
VARIABLE waterlevel : 0..3 ;  watertype
VARIABLE frametime : real              # this command's duration, in seconds
CONSTANT player_mins = (-16,-16,-24) ;  player_maxs = (16,16,32)
CONSTANT stepsize = 18
```

**Invariants** — the player's box is the **same hard-coded size** the map compiler precompiled a hull for
([`model.c`](../server/model.c.md)). It cannot be changed without changing the map format.

Immersion is a **level from zero to three**, not a boolean: feet wet, waist deep, eyes under. Each level changes different
behaviour, and the distinction is what makes wading, swimming and drowning three states.

## `PlayerMove`

**Contract** — runs one command: convert the command's duration to seconds, derive the view basis from the command's angles,
classify the position, handle the jump button and the water-jump check, then dispatch to the water, air or ground mover,
classify again, and clamp the player back onto the eighth-unit grid.

```text
FUNCTION player_move()
  frametime = cmd.msec / 1000
  numtouch = 0
  IF the player is a spectator  spectator_move() ;  RETURN
  derive forward/right/up from cmd.angles
  nudge_position()                     # snap to the wire grid, unstick if needed
  catagorize_position()                # onground, waterlevel, watertype
  IF eyes are under a liquid  check whether the player just went under
  IF the jump button is newly down  jump_button()
  ELSE IF it is up  clear the jump-held flag
  friction()
  IF waterlevel >= 2  water_move()
  ELSE               air_move()        # handles both ground and air
  catagorize_position()                # again: the move may have changed both
```

**Invariants** —

- **The command's duration drives the integration, not the frame time.** So a client at any frame rate and a server at any
  frame rate compute the same movement from the same command, which is the precondition for prediction agreeing with the
  authority. Every use of time in this file is the command's duration.
- **The position is classified twice**, before and after the move, because the ground state feeds the move and the move
  changes the ground state. Classifying once gives a player who cannot jump on the frame they land.
- **The jump button is edge-triggered**, and holding it must not bounce. The held flag lives in the movement state
  ([`pmove.h`](pmove.h.md)) precisely so both sides agree about it.

## `PM_Friction`

**Contract** — reduces speed each step: nothing while a water jump is in progress; on ground, a deceleration proportional to
speed but never below a floor value, and no friction at all if the ground under the player's leading edge is a ledge; in
water, a weaker proportional deceleration.

```text
FUNCTION friction()
  IF a water jump is in progress  RETURN
  speed = |velocity|
  IF speed < 1  zero the horizontal velocity ;  RETURN
  friction = the tunable
  IF on ground
    check the point just ahead and below the player
    IF it is not solid  friction *= 2         # a ledge: MORE friction, to stop at edges
    control = MAX(speed, stopspeed)
    drop = control * friction * frametime
  IF in water
    drop += speed * waterfriction * waterlevel * frametime
  newspeed = MAX(0, speed - drop) / speed
  velocity *= newspeed
```

**Invariants** —

- **Friction is proportional to speed but with a floor.** Below the floor speed, deceleration is constant, which is what
  brings a player to a stop in finite time instead of asymptotically. Pure proportional friction never stops anyone, and that
  is the bug this constant prevents.
- The **ledge case doubles friction**, so a player walking off a platform decelerates hard rather than sliding off. It is a
  playability decision, not physics.
- Friction acts on the **whole velocity including vertical** in water, and on horizontal only on ground — which follows from
  the ground mover zeroing the vertical component first.

## `PM_Accelerate`, `PM_AirAccelerate`

**Contract** — add speed along the desired direction, up to a desired speed, at a given rate; and the same with the desired
speed clamped to a small value for the purpose of the limit only.

```text
FUNCTION accelerate(wishdir, wishspeed, accel)
  IF dead or water-jumping  RETURN
  currentspeed = velocity . wishdir              # the PROJECTED speed
  addspeed = wishspeed - currentspeed
  IF addspeed <= 0  RETURN
  accelspeed = MIN(accel * frametime * wishspeed, addspeed)
  velocity += accelspeed * wishdir

FUNCTION air_accelerate(wishdir, wishspeed, accel)
  IF dead or water-jumping  RETURN
  wishspd = MIN(wishspeed, 30)                   # the clamp is on the LIMIT only
  currentspeed = velocity . wishdir
  addspeed = wishspd - currentspeed
  IF addspeed <= 0  RETURN
  accelspeed = MIN(accel * wishspeed * frametime, addspeed)   # full wishspeed here
  velocity += accelspeed * wishdir
```

**Invariants** —

- **The speed limit applies to the component of velocity along the desired direction, not to the speed.** So a player already
  moving fast in one direction may still accelerate freely in any direction perpendicular to it — the projection of their
  velocity onto that direction is near zero, so the limit does not bind. Total speed then grows without bound while the
  player turns.
- **In the air, the limit is clamped to a small value but the acceleration amount is not.** That asymmetry is what turns the
  above from a theoretical curiosity into a technique: in the air, turning slightly while holding a strafe key adds speed
  every step, and the result is the movement style the game became known for. Nobody designed it.
- **This is a deliberate rebuild decision, not an accident to be fixed.** A rebuild that clamps total speed instead produces
  a game that feels correct and plays differently, and its players will notice within minutes. The recipe's position: if you
  are rebuilding *this* game, reproduce both the projection and the asymmetric clamp exactly; if you are building a new one,
  know that this is the line you are choosing to cross.
- A dead player does not accelerate but does still slide, which is why the check is here and not in the caller.

## `PM_AirMove`

**Contract** — builds the desired velocity from the command's forward and side amounts along the view basis flattened to
horizontal, clamps it to the maximum speed, then either accelerates on the ground and runs the ground mover, or accelerates
in the air and runs the slide mover with gravity applied.

**Invariants** — the desired direction is **flattened to horizontal**, so looking up does not make the player fly. The
vertical component of the command is ignored entirely on ground and in air, and used only in water.

The maximum speed clamp scales the whole desired velocity rather than truncating it, so a diagonal input is not faster than a
straight one — the classic bug this avoids.

## `PM_FlyMove`

**Contract** — sweeps the player along its velocity for the remaining time, and on each impact clips the velocity into the
plane and continues, up to four times. Records every entity touched. Reports which axes were blocked.

```text
FUNCTION fly_move() -> blocked_flags
  collect up to 4 impact planes
  time_left = frametime
  REPEAT up to 4 times
    trace = sweep the player box from origin along velocity * time_left
    IF it started solid  zero the velocity ;  RETURN blocked
    IF any distance was covered  move to the impact point ;  remember it
    IF the sweep completed  BREAK
    record the touched entity
    IF the impact plane is shallow      mark blocked-by-floor
    IF the impact plane is vertical     mark blocked-by-wall
    time_left -= time_left * fraction
    add the plane to the list
    FOR EACH remembered plane
      clip the velocity into it
      IF the clipped velocity still moves into another remembered plane
        clip into the CROSS PRODUCT of the two planes    # slide along the crease
        IF that still moves into a third plane  zero the velocity   # a corner
```

**Invariants** — this is the same slide-move as the original's ([`sv_phys.c`](../../WinQuake/sv_phys.c.md)) and the
same reasoning applies: **clipping into one plane can push the mover into another**, so all remembered planes must be
rechecked, and two planes that both reject the motion leave only their crease to slide along. Three leaves nothing, and the
velocity must be zeroed or the player tunnels out of a corner.

The four-iteration cap is what bounds the work; the cost is that a very complex corner stops the player short for one step.

## `PM_ClipVelocity`

**Contract** — removes the component of a velocity that points into a plane, with a small overshoot, and reports nothing.

**Invariants** — the overshoot — clipping slightly *past* the plane rather than exactly onto it — is what keeps the mover from
re-colliding with the same surface on the next iteration due to rounding. The same epsilon reasoning as
[`world.c`](../server/world.c.md).

## `PM_GroundMove`

**Contract** — moves along the ground: try the whole move flat; if blocked, compute both a plain slide and a slide performed
after stepping up, then **keep whichever covered more horizontal distance**.

```text
FUNCTION ground_move()
  velocity.z = 0
  IF the velocity is zero  RETURN
  try moving straight to origin + velocity * frametime, at the same height
  IF nothing was hit  accept it ;  RETURN
  remember the original origin and velocity
  fly_move()                            # the plain slide
  remember its result as "down"
  restore the original origin and velocity
  sweep upward by the step height and accept whatever room there was
  fly_move()                            # the slide from up there
  sweep back down by the step height
  IF the surface landed on is too steep  use the "down" result
  accept the downward sweep's end as "up"
  IF "down" covered more horizontal distance than "up"
    use "down", including its velocity
  ELSE
    use "up", but take the VERTICAL velocity from "down"
```

**Invariants** —

- **Stairs are climbed by trying the move twice and keeping the better outcome.** There is no stair detection, no step
  geometry, no ramp classification. That is the whole mechanism, and it is why the player walks up stairs, over small
  obstacles and onto ledges with one piece of code.
- **The vertical velocity always comes from the plain slide**, even when the stepped-up result is used. Otherwise the upward
  sweep would give the player upward velocity and stepping onto a stair would launch them.
- If the stepped-up move lands on a steep surface, it is **rejected outright** — otherwise the player could step up onto a
  wall's sloped base and climb it.
- The comparison is **horizontal distance squared**, so no square root, and vertical progress does not count — a step up that
  goes nowhere horizontally loses to sliding along the wall.

## `PM_WaterMove`

**Contract** — builds the desired velocity from all three command axes along the full view basis, with a slow downward drift
when no input is given, clamps to a reduced maximum, accelerates at the water rate, then sweeps — first attempting to step out
onto land if the move is blocked and the player is near the surface.

**Invariants** — a player with no input **sinks slowly** rather than floating, which is a game decision. Swimming uses the
unflattened view direction, so a player swims where they look — the one place vertical aim moves the player.

## `PM_CatagorizePosition`

**Contract** — determines whether the player is on ground and how deep in liquid they are.

```text
FUNCTION catagorize_position()
  IF vertical velocity > 180  not on ground        # rising fast: skip the test
  ELSE
    sweep the player box one unit down
    IF the surface is too steep  not on ground
    ELSE  on ground, on that entity
    IF on ground  clear the water-jump timer, and snap to the impact point
    IF the ground is not the world  record it as touched
  waterlevel = 0
  IF the point just above the player's feet is in liquid
    waterlevel = 1 ;  watertype = that liquid
    IF the player's midpoint is in liquid   waterlevel = 2
    IF the point at eye height is in liquid waterlevel = 3
```

**Invariants** —

- **A fast-rising player is not on ground without testing**, which saves the sweep on every frame of a jump and, more
  importantly, prevents a player who has just jumped from immediately being re-grounded.
- **Ground is a slope test, not a contact test**: a surface steeper than the threshold is not ground, so the player slides
  down it. The threshold is the same value used throughout the engine and it defines what counts as a walkable slope.
- **Immersion is sampled at three heights** — just above the feet, the midpoint, and eye height — giving the three levels. The
  eye-height sample is what triggers drowning, and the midpoint sample is what switches from walking to swimming.
- Snapping the origin to the ground contact point each step is what keeps a walking player from drifting off surfaces.

## `JumpButton`

**Contract** — on a newly pressed jump: if dead, do nothing but mark it held; if a water jump is in progress, count it down;
if swimming, set an upward speed depending on the liquid; if in the air, do nothing; otherwise leave the ground with a fixed
upward speed.

**Invariants** —

- **Jump height is a fixed velocity added to whatever vertical velocity exists**, not a set velocity. So jumping while already
  moving upward — off a moving platform, or out of a ramp — goes higher. That is a feature players exploit.
- The **hold flag prevents repeated jumping while the key is down**, which the source names as the reason. Removing it gives a
  player who bounces continuously.
- Swimming upward speed **differs by liquid**, water fastest and lava slowest.

## `CheckWaterJump`

**Contract** — when the player is deep in liquid and facing a ledge just above the surface, give them an upward and forward
impulse and start a timer during which normal movement is suspended.

**Invariants** — the timer is what makes the manoeuvre work: friction and acceleration are **both disabled while it runs**, so
the impulse is not immediately damped away. That is why the timer is checked at the top of three separate functions, and a
rebuild must check it in all of them.

It refuses to trigger immediately after entering the water, so diving in does not bounce the player straight out.

## `NudgePosition`

**Contract** — snaps the origin to the eighth-unit grid, then, if the player does not fit there, tries the 27 neighbouring
grid points and takes the first that fits. Leaves the original if none does.

**Invariants** —

- **The snap to the eighth-unit grid is mandatory, not cosmetic.** The wire format transmits coordinates at that resolution
  ([`common.c`](common.c.md)), so the server's position and the client's prediction can only agree if both round to the same
  grid. Without the snap, prediction and authority diverge slowly and the player twitches. *This four-line function is what
  makes prediction stable.*
- The neighbour search exists because rounding can put a player a fraction inside a wall. Trying the neighbours in a fixed
  order — zero, then negative, then positive on each axis — means both sides make the **same** choice, which matters as much
  as the choice being valid.

## `SpectatorMove`

**Contract** — frictionless flying movement with its own maximum speed, no collision, along the full view basis.

**Invariants** — a spectator does not collide with anything, which is why this is a separate function rather than a mode of
the others. It reads a recorded server version to stay compatible with servers that used a different spectator speed — the one
piece of version negotiation in the movement code.

## `Pmove_Init`

**Contract** — builds the fabricated box hull the collision queries use ([`pmovetst.c`](pmovetst.c.md)).

**Notes** — the single most valuable property of this file is its **lack of dependencies**: it reads the movement record, the
tunables and nothing else. That is what let it be compiled into two programs, and it is the shape a rebuild should keep even
if it never plans to predict — pure movement is testable movement.
