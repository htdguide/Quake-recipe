# WinQuake/sv_phys.c

> The simulation: one dispatch over the movement types, a slide-along-planes mover that is the heart of all of them, and a set of documented hacks that keep entities from getting stuck in the collision hull.

**Needs** — [`server.h`](server.h.md) · [`world.h`](world.h.md) · [`progs.h`](progs.h.md) · [`quakedef.h`](quakedef.h.md) · [`mathlib.h`](mathlib.h.md) · [`cvar.h`](cvar.h.md) · [`pr_exec.c`](pr_exec.c.md) (callbacks) · [`pr_edict.c`](pr_edict.c.md) (optional field lookup) · [`sv_move.c`](sv_move.c.md) (the ground test) · [`sv_main.c`](sv_main.c.md) (sound) · [`bspfile.h`](bspfile.h.md) (contents values)
**Used by** — [`host.c`](host.c.md) calls the frame entry point; [`sv_user.c`](sv_user.c.md) reuses the movers; [`pr_cmds.c`](pr_cmds.c.md) reuses the ground drop
**Tier floor** — none; a simulation over floats

## Purpose

Everything that moves in the world moves through this file, once per server tick. It is worth
reading for three things.

The **mover** — the routine that slides an entity along up to four surfaces in one step,
including the crease where two meet — is the engine's core physics algorithm, and it is what
makes movement in this game feel the way it does.

The **movement type dispatch** is the closed set of behaviours the game can ask for
([`server.h`](server.h.md)), and each branch is a short, complete statement of one behaviour.

And the **hacks**. Four of them, each labelled as such in the source, each compensating for
the collision hull's finite precision, and each visible in play. They are not incidental:
delete them and players stick in doorways, fall through slopes, and freeze in corners. A
rebuild must implement all four, and the twin states what each is for so a rebuild can decide
whether its own collision system still needs them.

## State

```text
VARIABLE sv_friction    : Cvar = 4       # server-announced
VARIABLE sv_stopspeed   : Cvar = 100
VARIABLE sv_gravity     : Cvar = 800     # units per second squared;
                                         # server-announced
VARIABLE sv_maxvelocity : Cvar = 2000    # per axis
VARIABLE sv_nostep      : Cvar = 0       # disables stair climbing

CONSTANT move_epsilon     = 0.01
CONSTANT stop_epsilon     = 0.1          # velocity below this becomes zero
CONSTANT max_clip_planes  = 5
CONSTANT stepsize         = 18           # the tallest step a player climbs
```

**Invariants** — gravity is 800 units per second squared and a step is 18 units. Those two
numbers, together with the player's 56-unit height and the movement speeds in
[`sv_user.c`](sv_user.c.md), *are* the game's movement feel. Every map's geometry is built
around the 18-unit step in particular: stairs are 16 units, so a player climbs them, and
ledges are 24, so a player does not.

## The frame

### `SV_Physics`

**Contract** — the once-per-tick entry point. Calls the game's frame-start function, then
dispatches every live entity to its movement type's handler, then advances the simulation
clock. Honours the retouch countdown by relinking every entity, which fires every trigger
overlap. An unrecognized movement type is a fatal error.

```text
FUNCTION sv_physics()
  the globals self and other = the world ;  the global time = sv.time
  run the game's StartFrame function

  FOR EACH entity ent, index i, FROM 0 TO num_edicts-1
    IF ent IS free  CONTINUE
    IF the global force_retouch != 0
      sv_link_edict(ent, touch_triggers = true)   # even stationary entities

    IF i IS a player slot            sv_physics_client(ent, i)
    ELSE SELECT ent.movetype
      push        sv_physics_pusher(ent)
      none        sv_physics_none(ent)
      noclip      sv_physics_noclip(ent)
      step        sv_physics_step(ent)
      toss, bounce, fly, flymissile
                  sv_physics_toss(ent)
      otherwise   FAIL WITH "bad movetype <n>"

  IF the global force_retouch != 0  decrement it
  sv.time = sv.time + host_frametime
```

**Invariants** — the iteration is over the entity array **in index order**, from entity 0.
Order matters and is observable: an entity moved earlier in the sweep is at its new position
when a later one tests against it. A rebuild that parallelizes or reorders changes collision
outcomes.

Players are dispatched by *index*, not by movement type, so a player's movement type only
selects among the branches inside the player handler.

The retouch countdown relinks **everything**, including stationary entities, which is the
only way an entity already inside a trigger gets that trigger to fire. It is decremented once
per frame, not once per entity.

The clock advances **after** all movement, so every entity in one tick sees the same time.

**Notes** — a commented-out call to a whole-world sanity check sits in the loop's place. The
check ([`SV_CheckAllEnts`](#sv_checkallents)) is useful for a rebuild validating its
collision and is worth enabling behind a switch.

### `SV_CheckAllEnts`

**Contract** — reports any entity currently stuck inside solid, skipping the movement types
that are allowed to be. A diagnostic; not called.

### `SV_CheckVelocity`

**Contract** — takes an entity; replaces any not-a-number velocity or position component with
zero, reporting the entity's class, and clamps each velocity component into the maximum.

**Invariants** — per-axis clamping, not magnitude clamping, so the maximum diagonal speed is
larger than the maximum axial speed by a factor of the square root of three. Observable and
exploited by players.

The not-a-number check exists because game logic can divide by zero
([`pr_exec.c`](pr_exec.c.md)) and the result must not reach the collision system, where it
would make every comparison false and the sweep's behaviour undefined.

### `SV_RunThink`

**Contract** — takes an entity; if its scheduled think time has arrived within this tick, runs
its think callback with the clock set to the *scheduled* time rather than the current one.
Returns whether the entity still exists afterwards.

```text
FUNCTION sv_run_think(ent) -> bool
  t = ent.nextthink
  IF t <= 0 OR t > sv.time + host_frametime  RETURN true    # not due yet
  IF t < sv.time  t = sv.time            # never run in the past, even though
                                         # a local-time entity can schedule so
  ent.nextthink = 0                      # cleared BEFORE the callback, so the
                                         # callback may reschedule
  the global time = t
  the global self = ent ;  the global other = the world
  run ent.think to completion
  RETURN NOT ent.free
```

**Invariants** — the schedule is cleared *before* the callback, which is what lets a think
function set its own next think. A callback that does not is never called again.

The clock is set to the **scheduled** time, not the current one, so a think function's sense
of "now" is when it was supposed to run. That is what keeps an animation chain
([`pr_exec.c`](pr_exec.c.md#state)) from drifting: each frame's think schedules the next at
its own nominal time plus a tenth of a second, and the drift does not accumulate.

The "don't run in the past" clamp exists because a moving-platform entity runs on its own
local clock and can schedule a think behind the simulation.

Returning whether the entity survives is essential: a think function commonly removes its own
entity, and every caller checks.

### `SV_Impact`

**Contract** — takes two entities that have touched; runs each one's touch callback with
itself as the acting entity and the other as the partner, skipping either whose callback is
absent or which is non-solid. Saves and restores the acting-entity globals.

**Invariants** — **both** callbacks run, in order, and the first may free either entity. The
second call is made without re-checking, so a touch function that removes its partner leaves
the second call operating on a freed entity — which is safe only because freeing clears the
callback ([`pr_edict.c`](pr_edict.c.md#ed_free)). A rebuild must clear the callback on free or
must re-check here.

---

## The mover

### `ClipVelocity`

**Contract** — takes a velocity, a surface normal and an overbounce factor; returns the
velocity with its component into the surface removed and scaled, and a code saying whether
the surface was a floor or a vertical wall. Components smaller than a tenth of a unit per
second become exactly zero.

```text
FUNCTION clip_velocity(in, normal, overbounce) -> (out, blocked)
  blocked = 0
  IF normal[2] >  0  blocked = blocked BITOR 1      # a floor
  IF normal[2] == 0  blocked = blocked BITOR 2      # a vertical wall
  backoff = dot(in, normal) * overbounce
  FOR EACH axis i
    out[i] = in[i] - normal[i]*backoff
    IF |out[i]| < 0.1  out[i] = 0                   # kill residual jitter
  RETURN (out, blocked)
```

**Invariants** — an overbounce of 1 slides exactly along the surface; greater than 1 pushes
*away* from it, which is how bouncing is implemented (1.5 for a grenade). The value is a
caller's choice, not a material property.

The tenth-of-a-unit cutoff is what stops an entity resting on a slope from jittering forever
on residual velocity. Without it the slide computation leaves a tiny component every frame.

The floor test is "the normal has any upward component", which is *not* the same as the
0.7 threshold used elsewhere to decide whether a surface is walkable. Two different notions
of floor, deliberately.

### `SV_FlyMove`

**Contract** — takes an entity, a duration and an optional place to record a wall hit; moves
the entity along its velocity for that duration, sliding along up to four surfaces and along
the crease where two meet. Returns a bit code: 1 if it touched a floor, 2 if it touched a
vertical wall, and the values 3 and 7 for the two dead-stop cases. Runs touch callbacks at
every impact.

```text
FUNCTION sv_fly_move(ent, time, steptrace) -> int
  numbumps = 4
  blocked = 0
  original_velocity = primal_velocity = ent.velocity
  numplanes = 0
  time_left = time

  REPEAT numbumps TIMES
    IF ent.velocity IS zero  BREAK
    end = ent.origin + ent.velocity * time_left
    trace = sweep ent's box FROM ent.origin TO end, excluding ent

    IF trace.allsolid                       # trapped inside something
      ent.velocity = zero ;  RETURN 3

    IF trace.fraction > 0                   # some ground was covered
      ent.origin = trace.endpos
      original_velocity = ent.velocity
      numplanes = 0                         # a clean step resets the plane set

    IF trace.fraction == 1  BREAK           # completed the move
    IF trace.ent IS nothing  FAIL WITH "SV_FlyMove: !trace.ent"

    IF trace.plane.normal[2] > 0.7          # a walkable floor
      blocked = blocked BITOR 1
      IF trace.ent IS map geometry
        set ent's on-ground flag ;  ent.groundentity = trace.ent
    IF trace.plane.normal[2] == 0           # a vertical wall
      blocked = blocked BITOR 2
      IF steptrace WANTED  record this trace

    sv_impact(ent, trace.ent)               # arbitrary game code
    IF ent IS free  BREAK

    time_left = time_left - time_left * trace.fraction

    IF numplanes >= 5                       # "shouldn't really happen"
      ent.velocity = zero ;  RETURN 3
    planes[numplanes] = trace.plane.normal ;  numplanes = numplanes + 1

    # --- resolve the velocity against EVERY plane hit so far ---
    # Try each plane in turn: slide along it and check the result does not
    # go into any of the others.
    FOR i FROM 0 TO numplanes-1
      candidate = clip_velocity(original_velocity, planes[i], overbounce = 1)
      IF candidate goes into no OTHER plane  BREAK              # accept it
    IF such an i was found
      ent.velocity = candidate
    ELSE
      # No single plane works: move along the CREASE between exactly two.
      IF numplanes != 2
        ent.velocity = zero ;  RETURN 7      # three-plane corner: dead stop
      dir = cross(planes[0], planes[1])
      ent.velocity = dir SCALED BY dot(dir, ent.velocity)

    # If the resolved velocity opposes where we started, stop dead rather
    # than oscillate in a sloping corner.
    IF dot(ent.velocity, primal_velocity) <= 0
      ent.velocity = zero ;  RETURN blocked

  RETURN blocked
```

**Invariants** — this is the routine to get exactly right, and five details carry it.

**Four attempts, not one.** An entity that hits a wall slides and tries again, up to four
times per tick. Fewer attempts and a player sliding into a corner stops short of it.

**The plane set resets on progress.** Any step that actually covered distance clears the
accumulated planes, because they described an obstacle the entity is no longer against. Not
resetting makes movement seize up after a few frames of wall contact.

**The velocity is re-derived from the pre-bump velocity**, not from the already-clipped one.
So two sequential clips do not compound into an over-correction.

**The crease case is the interesting one.** When no single surface admits the motion, the
entity moves along the line where two surfaces meet — the cross product of their normals,
scaled by how much of the velocity points along it. That is what lets a player slide smoothly
down the join between a wall and a ramp instead of stopping. With three or more planes there
is no crease and the entity stops dead.

**The reversal check** stops an entity whose resolved velocity now opposes its original
direction. Without it, an entity in a converging corner oscillates between two clip solutions
forever, which is visible as a buzzing jitter.

The **0.7 threshold** for "this is a floor I can stand on" is about 45 degrees and is the
game's definition of a climbable slope. A different value changes which ramps players can
walk up, across every map.

Note the impact callbacks run **inside** the bump loop, so game code executes between the
sub-steps of one entity's move and may free the entity, change its velocity, or move it.
The loop checks only for the entity being freed.

### `SV_AddGravity`

**Contract** — subtracts one tick's worth of gravity from an entity's vertical velocity,
scaled by a per-entity multiplier if the loaded game logic declares one.

```text
FUNCTION sv_add_gravity(ent)
  # The multiplier is an OPTIONAL field: this build looks it up by name,
  # because the compiled-in layout does not include it.
  val = the value of ent's "gravity" field, IF the game declared one
  multiplier = val IF it exists and is non-zero ELSE 1.0
  ent.velocity[2] = ent.velocity[2] - multiplier * sv_gravity * host_frametime
```

**Invariants** — the per-entity multiplier is reached through the run-time field lookup
([`pr_edict.c`](pr_edict.c.md#getedictfieldvalue)), which is the only place in the physics
that consults an optional field. It is how game modifications add low-gravity items without
an engine change, and it is why the lookup cache exists.

---

## Pushing entities

### `SV_PushEntity`

**Contract** — moves an entity by a displacement, sweeping and stopping at the first
obstacle, relinking it with triggers fired, and running the impact callbacks. **Does not touch
the velocity.** Returns the trace. The trace mode depends on the entity: flying missiles get
the enlarged monster box, triggers and non-solid entities collide only with map geometry, and
everything else collides normally.

**Invariants** — the trigger-and-non-solid case colliding only with geometry is what lets a
dropped item fall through a monster and land on the floor.

### `SV_PushMove`

**Contract** — takes a pushing entity and a duration; moves it, then moves everything it now
overlaps or that is standing on it by the same displacement. If any of those cannot be moved
out of the way, undoes the whole operation — the pusher and every entity already moved — and
runs the pusher's blocked callback.

```text
FUNCTION sv_push_move(pusher, movetime)
  IF pusher.velocity IS zero
    pusher.ltime = pusher.ltime + movetime ;  RETURN

  move = pusher.velocity * movetime
  final_box = pusher's world box, DISPLACED BY move
  pushorig = pusher.origin

  pusher.origin = pusher.origin + move          # move the pusher FIRST
  pusher.ltime = pusher.ltime + movetime
  sv_link_edict(pusher, touch_triggers = false)

  moved = empty list
  FOR EACH live entity check
    IF check.movetype IS one OF { push, none, noclip }  CONTINUE

    # An entity standing ON the pusher always moves. Otherwise it only moves
    # if it is now inside the pusher's new position.
    IF NOT (check is on the ground AND its ground entity IS pusher)
      IF check's box does not overlap final_box  CONTINUE
      IF NOT check is stuck                      CONTINUE

    IF check.movetype IS NOT walk
      clear check's on-ground flag               # players keep theirs

    record (check, check.origin) IN moved
    pusher.solid = not-solid                     # so the push does not
    sv_push_entity(check, move)                  # collide with the pusher
    pusher.solid = solid_bsp

    IF check is STILL stuck
      IF check has zero width           CONTINUE   # a point: ignore it
      IF check.solid IS not-solid OR trigger
        # a corpse: SHRINK IT TO NOTHING rather than block the door
        check.mins = check.maxs = zero ;  CONTINUE

      # --- the push is blocked: undo everything ---
      check.origin = its recorded position
      sv_link_edict(check, touch_triggers = true)
      pusher.origin = pushorig
      sv_link_edict(pusher, touch_triggers = false)
      pusher.ltime = pusher.ltime - movetime
      IF pusher has a blocked callback
        the global self = pusher ;  the global other = check
        run pusher.blocked to completion
      FOR EACH (entity, position) IN moved
        entity.origin = position ;  relink it
      RETURN
```

**Invariants** — several decisions here are visible in play.

**An entity standing on the pusher is moved unconditionally**, without an overlap test. That
is how a platform carries a player upward: the player is not inside the platform, they are on
top of it.

**The pusher is made non-solid while pushing each entity.** Otherwise the sweep would
immediately hit the pusher the entity is being pushed out of.

**A corpse in a doorway is crushed to a point rather than blocking it.** The code detects a
non-solid or trigger entity that cannot be moved and sets its bounding box to zero. That is
why dead bodies never jam doors in this game, and it permanently destroys the corpse's size.

**A zero-width entity is ignored** rather than blocking.

**The undo is complete and ordered**: the blocked entity, the pusher, the pusher's local
clock, then every entity already moved, in the order they were moved. Partial undo leaves
entities inside walls.

**Notes** — the moved-entity arrays are sized to the entity maximum and are **stack-allocated
locals**, which is 600 entity pointers and 600 vectors — about 9 kilobytes of stack per call,
and the calls do not nest. A rebuild should allocate them once.

The pusher's own clock is separate from the simulation clock, which is what lets a platform
advance in sub-tick increments to hit its think time exactly.

### `SV_Physics_Pusher`

**Contract** — advances a pushing entity, splitting the tick if its think time falls inside
it so that the think runs at exactly the right position, then runs the think if it became due.

```text
FUNCTION sv_physics_pusher(ent)
  oldltime = ent.ltime
  thinktime = ent.nextthink
  IF thinktime < ent.ltime + host_frametime
    movetime = max(thinktime - ent.ltime, 0)     # advance only up to the think
  ELSE
    movetime = host_frametime

  IF movetime != 0  sv_push_move(ent, movetime)  # advances ltime unless blocked

  IF thinktime > oldltime AND thinktime <= ent.ltime
    ent.nextthink = 0
    the global time = sv.time
    the global self = ent ;  the global other = the world
    run ent.think to completion
```

**Invariants** — the tick is **truncated** at the think time rather than split into two
moves, so a platform that should stop exactly at its destination advances only to there this
tick and the remainder of the tick is simply not simulated for it. That is why platforms in
this game stop precisely and why they appear to pause for a fraction of a frame at each end.

The think is gated on the local clock having crossed the think time, which it has not if the
push was blocked — so a blocked platform does not run its arrival callback.

The think's `self` sees the *simulation* clock, not the local one.

---

## Player movement

### `SV_CheckStuck`

**Contract** — takes an entity that may be inside solid; tries to free it, first by returning
it to its last known good position, then by searching a small region around its current one.
Records the current position as known-good when it is not stuck.

```text
FUNCTION sv_check_stuck(ent)
  IF NOT stuck
    ent.oldorigin = ent.origin ;  RETURN         # remember a good position

  org = ent.origin
  ent.origin = ent.oldorigin                     # try the last good spot
  IF NOT stuck
    print "Unstuck." ;  relink with triggers ;  RETURN

  # Search a 3 x 3 x 18 lattice around the original position, ONE UNIT apart,
  # rising: horizontal offsets -1, 0, +1 and vertical 0 through 17.
  FOR z FROM 0 TO 17
    FOR i IN {-1, 0, 1}
      FOR j IN {-1, 0, 1}
        ent.origin = org + (i, j, z)
        IF NOT stuck
          print "Unstuck." ;  relink with triggers ;  RETURN

  ent.origin = org
  print "player is stuck."
```

**Invariants** — the first of the file's four labelled hacks, and the source calls it "a big
hack to try and fix the rare case of getting stuck in the world clipping hull". The vertical
range of 18 units is exactly the step height, so the search can lift a player by a full step.
The horizontal range is one unit in each direction, matching the collision system's epsilon
([`world.c`](world.c.md)).

The iteration order matters: vertical outermost, so the **lowest** freeing position wins,
which keeps a freed player as close to the floor as possible.

The known-good position is only recorded when not stuck, so it is genuinely a last-good rather
than a last position.

### `SV_CheckWater`

**Contract** — takes an entity; determines how deeply it is submerged and in what, by sampling
three heights. Returns whether it is submerged past the middle.

```text
FUNCTION sv_check_water(ent) -> bool
  ent.waterlevel = 0 ;  ent.watertype = empty
  sample at ent.origin + (0, 0, ent.mins[2] + 1)        # just above the feet
  IF that content IS a liquid
    ent.watertype = it ;  ent.waterlevel = 1
    sample at the vertical MIDPOINT of the box
    IF liquid
      ent.waterlevel = 2
      sample at ent.origin[2] + ent.view_ofs[2]          # the EYES
      IF liquid  ent.waterlevel = 3
  RETURN ent.waterlevel > 1
```

**Invariants** — three discrete levels — feet, waist, eyes — not a continuous depth. Level 2
is where swimming physics begins; level 3 is where drowning does. The eye sample uses the
entity's own view offset, so a crouching entity drowns sooner.

The liquid test is "contents at or below water", which relies on the contents numbering being
ordered ([`bspfile.h`](bspfile.h.md)).

### `SV_WallFriction`

**Contract** — takes an entity and a wall it hit; reduces its horizontal velocity in
proportion to how directly it is facing that wall.

```text
FUNCTION sv_wall_friction(ent, trace)
  forward = the forward vector OF ent.v_angle
  d = dot(trace.plane.normal, forward) + 0.5
  IF d >= 0  RETURN                    # not facing into the wall enough
  # Remove the component into the wall, then scale what remains by (1 + d),
  # which is between 0 and 0.5 — so facing the wall dead-on stops you.
  into = the component of ent.velocity along the normal
  side = ent.velocity - into
  ent.velocity[0] = side[0] * (1 + d)
  ent.velocity[1] = side[1] * (1 + d)
```

**Invariants** — the friction depends on **where the player is looking**, not on where they
are moving. Facing a wall directly gives a normal-dot-forward of −1, so `d` is −0.5 and the
remaining velocity is halved; facing 60 degrees away gives `d` of 0 and no friction at all.
That is a deliberate feel decision: running along a wall while looking ahead is fast, and
scraping into it while staring at it is slow.

Only the horizontal components are scaled, so vertical motion is untouched.

### `SV_TryUnstick`

**Contract** — takes an entity that has come to a dead stop and its pre-move velocity; tries
pushing it two units in each of eight horizontal directions and retrying the move, accepting
the first attempt that makes more than four units of horizontal progress. Zeroes the velocity
if none does.

```text
FUNCTION sv_try_unstick(ent, oldvel) -> int
  oldorg = ent.origin
  FOR EACH dir IN the eight 2-unit horizontal nudges
      # +x, +y, -x, -y, then the four diagonals
    sv_push_entity(ent, dir)
    ent.velocity = (oldvel[0], oldvel[1], 0)
    clip = sv_fly_move(ent, 0.1, record a step trace)
    IF the horizontal displacement FROM oldorg exceeds 4 units
      RETURN clip                                  # it worked
    ent.origin = oldorg                            # put it back and try again
  ent.velocity = zero
  RETURN 7                                         # still stuck
```

**Invariants** — the second labelled hack, and the source is explicit about why: limited float
precision at some angle joins in the collision hull. The retry duration is a hard-coded
**0.1 seconds**, not the frame time, so the test move is longer than a real tick — which is
what makes four units of progress a meaningful threshold.

The eight directions are tried in a fixed order — axial first, then diagonal — so the outcome
is deterministic.

The nudges are **not** undone on success: the entity keeps the two-unit displacement that
freed it. That is the hack's visible symptom, a small sideways jump when a player unsticks
from a doorframe.

### `SV_WalkMove`

**Contract** — takes a player entity; performs a slide move, and if that move was blocked by a
vertical wall, retries it as a step: raise by the step height, move forward, lower back down,
and keep whichever result ended on walkable ground.

```text
FUNCTION sv_walk_move(ent)
  oldonground = ent's on-ground flag
  clear ent's on-ground flag
  oldorg = ent.origin ;  oldvel = ent.velocity

  clip = sv_fly_move(ent, host_frametime, record a step trace)
  IF clip LACKS the wall bit  RETURN              # nothing to step over

  IF NOT oldonground AND ent.waterlevel == 0  RETURN   # no stairs while jumping
  IF ent.movetype IS NOT walk                 RETURN   # changed mid-move
  IF sv_nostep                                RETURN
  IF ent has the water-jump flag              RETURN

  nosteporg = ent.origin ;  nostepvel = ent.velocity   # keep the plain result

  # --- retry as a step ---
  ent.origin = oldorg
  upmove   = (0, 0, +18)
  downmove = (0, 0, -18 + oldvel[2] * host_frametime)   # note the vertical term
  sv_push_entity(ent, upmove)

  ent.velocity = (oldvel[0], oldvel[1], 0)
  clip = sv_fly_move(ent, host_frametime, record a step trace)

  IF clip != 0 AND the horizontal displacement FROM oldorg is under 1/32 unit
    clip = sv_try_unstick(ent, oldvel)           # stepping made no progress

  IF clip HAS the wall bit  sv_wall_friction(ent, the step trace)

  downtrace = sv_push_entity(ent, downmove)

  IF downtrace.plane.normal[2] > 0.7
    IF ent.solid IS solid_bsp
      set ent's on-ground flag ;  ent.groundentity = downtrace.ent
  ELSE
    # The step landed somewhere unwalkable — a wall-slope junction. Discard
    # the step and use the plain slide result instead.
    ent.origin = nosteporg ;  ent.velocity = nostepvel
```

**Invariants** — the whole routine is "try it flat; if a wall stopped you, try it as a step;
keep the better outcome", and each guard has a reason.

**No stairs while airborne**, unless in water. Otherwise a jumping player would be lifted onto
ledges they jumped at, which reads as sticking to walls.

The **downward move includes the tick's vertical velocity**, so a player who is also falling
descends by the step height *plus* the fall. Without that term a falling player would hover
one step above the floor.

The **step result is discarded if it did not land on walkable ground.** The source explains:
this happens near wall-and-slope combinations and would otherwise let a player hop up a slope
too steep to climb. So the fallback is not a safety net, it is a rule closing an exploit.

The **1/32-unit progress threshold** matches the collision epsilon exactly
([`world.c`](world.c.md)): making less than one epsilon of progress means the collision system
refused the move, not that the geometry blocked it, and the unstick hack is the response.

The on-ground flag is set only when standing on **map geometry**, never on another entity —
which is why a player standing on a monster is not "on the ground" and cannot jump from it.

The two labelled uncertainties in the source, both asking whether the vertical pushes need to
relink, are worth a rebuild's attention: relinking fires triggers, so a player stepping up
through a trigger fires it twice.

### `SV_Physics_Client`

**Contract** — takes a player entity and its slot; runs the game's pre-think, dispatches on
the movement type, relinks with triggers, and runs the game's post-think. Skips an
unoccupied slot. An unrecognized movement type is a fatal error.

```text
FUNCTION sv_physics_client(ent, num)
  IF the slot is not active  RETURN
  the global time = sv.time ;  the global self = ent
  run the game's PlayerPreThink

  sv_check_velocity(ent)
  SELECT ent.movetype
    none    IF NOT sv_run_think(ent)  RETURN
    walk    IF NOT sv_run_think(ent)  RETURN
            IF NOT sv_check_water(ent) AND NOT water-jumping
              sv_add_gravity(ent)
            sv_check_stuck(ent)
            sv_walk_move(ent)
    toss, bounce   sv_physics_toss(ent)
    fly     IF NOT sv_run_think(ent)  RETURN
            sv_fly_move(ent, host_frametime)
    noclip  IF NOT sv_run_think(ent)  RETURN
            ent.origin = ent.origin + ent.velocity * host_frametime
    otherwise  FAIL WITH "bad movetype <n>"

  sv_link_edict(ent, touch_triggers = true)
  the global time = sv.time ;  the global self = ent
  run the game's PlayerPostThink
```

**Invariants** — the pre-think runs **before** the movement command has any effect and the
post-think after, which is the contract the game's weapon and damage logic is built on. The
post-think runs even when the movement type is one that returned early from its think — no, it
does not: an early return from a think that removed the entity skips the post-think entirely.
A rebuild must keep that, because the game's post-think assumes a live player.

Gravity is skipped when the player is swimming *or* climbing out of water. The water-jump
state suspending gravity is what makes the climb-out motion work at all.

---

## The remaining movement types

### `SV_Physics_None`

**Contract** — runs the think and nothing else.

### `SV_Physics_Noclip`

**Contract** — runs the think, then advances position and angles by velocity and angular
velocity, relinking without firing triggers.

**Invariants** — relinking **without** triggers is why a player in this mode passes through
trigger volumes without activating them.

### `SV_CheckWaterTransition`

**Contract** — takes an entity; detects a crossing between air and liquid and plays a splash.
Records the new liquid type and a level of 1.

```text
FUNCTION sv_check_water_transition(ent)
  cont = the contents AT ent.origin
  IF ent.watertype IS unset                 # just spawned
    ent.watertype = cont ;  ent.waterlevel = 1 ;  RETURN
  IF cont IS a liquid
    IF ent.watertype WAS empty  play the splash sound       # entered
    ent.watertype = cont ;  ent.waterlevel = 1
  ELSE
    IF ent.watertype WAS NOT empty  play the splash sound   # left
    ent.watertype = empty
    ent.waterlevel = cont                   # see the note
```

**Invariants** — the level is sampled at the entity's **origin only**, not at three heights,
so a tossed object is either in liquid or not with no notion of depth. That is correct for
grenades and gibs, which is all that uses this.

**Notes** — the else branch assigns the *contents value* to the water level, which for air is
−1. Nothing reads a tossed entity's water level, so it is harmless; it is clearly a slip and a
rebuild should assign zero.

### `SV_Physics_Toss`

**Contract** — takes an entity using tossing, bouncing or flying movement; runs its think,
applies gravity unless flying, advances angles, moves along velocity stopping at the first
obstacle, and on impact either bounces or comes to rest. Does nothing at all if already at
rest on the ground.

```text
FUNCTION sv_physics_toss(ent)
  IF NOT sv_run_think(ent)  RETURN
  IF ent is on the ground   RETURN            # at rest: no work at all
  sv_check_velocity(ent)
  IF ent.movetype IS NOT fly AND NOT flymissile
    sv_add_gravity(ent)

  ent.angles = ent.angles + ent.avelocity * host_frametime
  trace = sv_push_entity(ent, ent.velocity * host_frametime)
  IF trace.fraction == 1  RETURN              # nothing hit
  IF ent IS free          RETURN              # the impact removed it

  backoff = 1.5 IF ent.movetype IS bounce ELSE 1.0
  ent.velocity = clip_velocity(ent.velocity, trace.plane.normal, backoff)

  IF trace.plane.normal[2] > 0.7              # landed on walkable ground
    IF ent.velocity[2] < 60 OR ent.movetype IS NOT bounce
      set ent's on-ground flag ;  ent.groundentity = trace.ent
      ent.velocity = zero ;  ent.avelocity = zero
  sv_check_water_transition(ent)
```

**Invariants** — the early return for an entity on the ground is what makes hundreds of
settled gibs and dropped items free: they are skipped entirely. The flag is only cleared when
something else disturbs them ([`SV_PushMove`](#sv_pushmove) clears it on non-players).

**The 60-unit rest threshold** is the bounce cutoff: a grenade whose upward velocity after a
bounce is under 60 units per second stops instead of bouncing again. Non-bouncing entities
stop unconditionally on any walkable surface.

Coming to rest **zeroes the angular velocity too**, which is why a spinning grenade stops
spinning the instant it settles.

The bounce factor of 1.5 means a grenade retains half its incoming normal velocity reversed —
not a physical restitution coefficient, a tuned number.

### `SV_Physics_Step`

**Contract** — takes a stepped entity; if it is not supported and not flying or swimming,
applies gravity and moves it, plays a landing sound if it hit hard, then runs the think and
checks the liquid transition. A supported entity does not move here at all — its movement comes
from [`sv_move.c`](sv_move.c.md) under the game's control.

```text
FUNCTION sv_physics_step(ent)
  IF ent has NONE OF { on-ground, flying, swimming }
    hitsound = ent.velocity[2] < sv_gravity * -0.1      # falling fast enough
    sv_add_gravity(ent)
    sv_check_velocity(ent)
    sv_fly_move(ent, host_frametime)
    sv_link_edict(ent, touch_triggers = true)
    IF ent is NOW on the ground AND hitsound
      play the landing sound
  sv_run_think(ent)
  sv_check_water_transition(ent)
```

**Invariants** — this is the whole of stepped physics in the engine: **free fall, and nothing
else.** A monster's horizontal movement is the game logic calling the navigation host
functions, which go through [`sv_move.c`](sv_move.c.md) and are refused if they would walk off
a ledge. That division — the engine handles falling, the game handles walking, and the engine
vetoes bad steps — is the design, and it is why monsters in this game never fall off ledges
but do fall when a floor is removed.

The landing sound threshold is a tenth of gravity, so about 80 units per second — a fall of
roughly four units. The sound name is hard-coded here, in the engine, which is a small
layering violation the game cannot override.

**Notes** — the flag read is a *stale* one: the entity is considered unsupported based on last
tick's flag, and the flag is set by the move that follows. So an entity that loses its support
falls one tick late. Imperceptible, and it is why the routine needs no separate support test.
