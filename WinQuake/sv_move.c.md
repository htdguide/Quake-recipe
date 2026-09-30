# WinQuake/sv_move.c

> Monster locomotion: a step that refuses to walk off a ledge, and an eight-direction chase that finds its way around obstacles with no pathfinding at all.

**Needs** — [`server.h`](server.h.md) · [`world.h`](world.h.md) · [`progs.h`](progs.h.md) · [`quakedef.h`](quakedef.h.md) · [`mathlib.h`](mathlib.h.md) · [`pr_cmds.c`](pr_cmds.c.md) (the yaw turn) · [`bspfile.h`](bspfile.h.md)
**Used by** — [`pr_cmds.c`](pr_cmds.c.md) exposes two of these to game logic as host functions; [`sv_phys.c`](sv_phys.c.md) uses the ground test
**Tier floor** — none

## Purpose

There is no navigation mesh, no waypoint graph and no path search anywhere in this engine. A
monster chasing a player takes one step at a time in one of eight compass directions, and this
file is the whole of it. Two ideas carry the behaviour.

**A step is vetoed, not corrected.** The game logic asks to move a distance along a heading;
the engine checks that the destination has floor under it and refuses otherwise. That single
veto is why monsters in this game never walk off ledges while players do, and it replaces an
entire class of navigation machinery.

**Obstacle avoidance is a fixed preference order with randomness.** When the direct route is
blocked, the monster tries the two axial components of the direction to its target, then its
previous heading, then all eight compass points in a randomly chosen rotation order, and only
then reverses. The randomness is what keeps two monsters in the same corridor from locking into
the same failure.

## State

```text
CONSTANT stepsize = 18          # must agree with sv_phys.c
CONSTANT di_nodir = -1          # "no preference on this axis"
VARIABLE c_yes, c_no : int      # counters; never reported
```

## `SV_CheckBottom`

**Contract** — takes an entity; reports whether its whole footprint is supported, allowing for
step-height variation. Answers cheaply when all four bottom corners sit directly on solid
world; otherwise traces downward from the centre and from each corner and requires every
corner's floor to be within one step height of the centre's.

```text
FUNCTION sv_check_bottom(ent) -> bool
  mins = ent.origin + ent.mins ;  maxs = ent.origin + ent.maxs

  # Fast path: if solid world lies one unit under all four bottom corners,
  # accept without tracing.
  FOR EACH of the four horizontal corners
    IF the contents one unit below that corner IS NOT solid  GOTO real_check
  RETURN true

real_check:
  # Trace down from the CENTRE of the bottom face, up to two step heights.
  start = the centre of the bottom face
  stop  = start, lowered BY 2 * stepsize
  trace = sweep a POINT FROM start TO stop, ignoring monsters, excluding ent
  IF trace.fraction == 1  RETURN false              # no floor within 36 units
  mid = bottom = trace.endpos[2]

  # Every corner's floor must be within one step height of the centre's.
  FOR EACH of the four horizontal corners
    trace = sweep a POINT FROM that corner DOWN TO the same stop height
    IF trace.fraction != 1 AND trace.endpos[2] > bottom  bottom = trace.endpos[2]
    IF trace.fraction == 1 OR mid - trace.endpos[2] > stepsize  RETURN false
  RETURN true
```

**Invariants** — the fast path is the common case on flat floors and avoids five sweeps. It is
correct because solid world directly beneath every corner means the footprint is fully
supported; it declines to answer when any corner is over an entity or over air, because the
cheap test cannot tell a step from a chasm.

The real check's rule is **relative, not absolute**: a corner over a 17-unit drop is fine, a
corner over a 19-unit drop is not. That is what lets monsters walk across stairs and rubble
while refusing ledges, and the threshold is the same 18 units the player's step height uses.

The downward reach is **two step heights**, so a corner more than 36 units above nothing fails
immediately.

The traces are point traces with monsters excluded, so a monster is not supported by another
monster.

**Notes** — the accumulated `bottom` value is computed and never used; the test compares
against the centre's height throughout. Dead code, and the comments about corners being within
16 units disagree with the code's 18. A rebuild should implement the code.

## `SV_movestep`

**Contract** — takes an entity, a displacement and whether to relink; attempts the move,
adjusting for slopes and stairs, and returns whether it succeeded. On failure the entity does
not move at all. Flying and swimming entities take a different path that has no stepping and
that adjusts height toward their target.

```text
FUNCTION sv_movestep(ent, move, relink) -> bool
  oldorg = ent.origin
  neworg = ent.origin + move

  # --- flying and swimming: no stepping, but hunt vertically ---
  IF ent has the flying OR swimming flag
    REPEAT 2 TIMES, attempt number i
      neworg = ent.origin + move
      enemy = ent.enemy
      IF i == 0 AND enemy IS NOT the world
        # nudge 8 units toward the enemy's height, but only on the FIRST try
        dz = ent.origin[2] - enemy.origin[2]
        IF dz > 40  neworg[2] = neworg[2] - 8
        IF dz < 30  neworg[2] = neworg[2] + 8
      trace = sweep ent's box FROM ent.origin TO neworg, excluding ent
      IF trace.fraction == 1
        IF ent is SWIMMING AND the destination is NOT liquid
          RETURN false                      # a fish may not leave the water
        ent.origin = trace.endpos
        IF relink  sv_link_edict(ent, touch_triggers = true)
        RETURN true
      IF enemy IS the world  BREAK          # no second attempt without a target
    RETURN false

  # --- walking: step up, then settle down ---
  neworg[2] = neworg[2] + stepsize          # start a step height high
  end = neworg, lowered BY 2 * stepsize
  trace = sweep ent's box FROM neworg DOWN TO end, excluding ent

  IF trace.allsolid  RETURN false
  IF trace.startsolid
    # the raised start is inside something; try from the unraised height
    neworg[2] = neworg[2] - stepsize
    trace = sweep AGAIN FROM neworg DOWN TO end
    IF trace.allsolid OR trace.startsolid  RETURN false

  IF trace.fraction == 1
    # No floor anywhere in the 36-unit sweep: this is walking off an edge.
    IF ent has the partial-ground flag
      # unless its floor was pulled out from under it, in which case let it fall
      ent.origin = ent.origin + move
      IF relink  sv_link_edict(ent, touch_triggers = true)
      clear ent's on-ground flag
      RETURN true
    RETURN false                            # walked off an edge: refuse

  # There is floor. Provisionally stand on it and check the whole footprint.
  ent.origin = trace.endpos
  IF NOT sv_check_bottom(ent)
    IF ent has the partial-ground flag
      # it is already unsupported and is trying to correct; allow the move
      IF relink  sv_link_edict(ent, touch_triggers = true)
      RETURN true
    ent.origin = oldorg                     # put it back; refuse
    RETURN false

  clear ent's partial-ground flag           # it is properly supported now
  ent.groundentity = trace.ent
  IF relink  sv_link_edict(ent, touch_triggers = true)
  RETURN true
```

**Invariants** — the walking path's shape is **step up first, then fall down**, which is how
one operation handles flat ground, stairs and slopes without distinguishing them: begin the
sweep a step height above the destination and let it settle. The downward reach of two step
heights is what allows descending a stair as well as climbing one.

The **partial-ground flag inverts the veto**. Normally a step onto unsupported ground is
refused. But an entity already flagged as partially unsupported — because a bridge was removed
under it — is allowed to move anyway, and even allowed to fall. Without that exception a
monster standing where its floor used to be could never move again, because every step would be
refused for the same reason. That flag, set by
[`SV_FixCheckBottom`](#sv_fixcheckbottom), is the escape hatch.

The **swimming restriction** — refuse a move whose destination is not liquid — is what keeps
fish in water without any special-case geometry.

The **flying height hunt** nudges 8 units toward the target's height when there is a target,
and retries without the nudge if the nudged move fails. The dead band between 30 and 40 units
of vertical separation is what stops a flying monster oscillating: inside it, neither branch
fires. That two-attempt structure is why flying monsters circle rather than dart.

The **start-solid retry** exists because raising the sweep's start by a step height can put it
inside a low ceiling; dropping back to the unraised height is the fallback.

## `SV_StepDirection`

**Contract** — takes an entity, a heading in degrees and a distance; turns the entity toward
that heading at its own turn rate, then attempts the step. If the step succeeded but the entity
has not yet turned within 45 degrees of the heading, the position is reverted while the turn is
kept. Always relinks. Returns whether the step succeeded.

```text
FUNCTION sv_step_direction(ent, yaw, dist) -> bool
  ent.ideal_yaw = yaw
  turn ent toward its ideal yaw at its own turn rate     # pr_cmds.c's changeyaw
  move = (cos(yaw)*dist, sin(yaw)*dist, 0)
  oldorigin = ent.origin
  IF sv_movestep(ent, move, relink = false)
    delta = ent.angles[YAW] - ent.ideal_yaw
    IF delta > 45 AND delta < 315
      ent.origin = oldorigin           # turned too little: keep the TURN,
                                       # discard the STEP
    sv_link_edict(ent, touch_triggers = true)
    RETURN true
  sv_link_edict(ent, touch_triggers = true)
  RETURN false
```

**Invariants** — this is why monsters turn before they walk. A monster whose turn rate is slow
spends several frames facing the new direction while reporting success, and only actually moves
once it is within 45 degrees. The reported success is deliberate: the caller must not treat
"still turning" as "blocked" and pick a different direction.

The angle test is written as a single range on an unwrapped difference, which covers both
signs because the difference is always in [0, 360) after the quantized turn
([`pr_cmds.c`](pr_cmds.c.md#pf_changeyaw-changeyaw-49)).

The relink happens on **both** paths and with triggers enabled, so a monster fires triggers
it steps into even when the step is then reverted. A rebuild reordering this changes when
triggers fire.

## `SV_FixCheckBottom`

**Contract** — sets an entity's partial-ground flag, marking it as standing somewhere its
footprint is not fully supported.

**Notes** — one line, and it is the only place the flag is set. Called when a monster has run
out of directions *and* is unsupported, which is the "the bridge was removed" case. From then
on [`SV_movestep`](#sv_movestep) lets it move and fall.

## `SV_NewChaseDir`

**Contract** — takes a monster, a target and a distance; picks a new heading and takes a step,
trying in a fixed order with two randomized choices. Sets the monster's heading back to its
previous one if nothing works, and flags it as unsupported if it also has no floor.

```text
FUNCTION sv_new_chase_dir(actor, enemy, dist)
  olddir     = actor.ideal_yaw, SNAPPED to a multiple of 45 degrees
  turnaround = olddir + 180                # the one direction to try LAST

  deltax = enemy.origin[0] - actor.origin[0]
  deltay = enemy.origin[1] - actor.origin[1]
  # Reduce each axis to a compass heading, with a 10-unit dead band.
  dx = 0    IF deltax >  10
       180  IF deltax < -10
       no-preference OTHERWISE
  dy = 270  IF deltay < -10
       90   IF deltay >  10
       no-preference OTHERWISE

  # 1. Both axes have a preference: try the DIAGONAL between them.
  IF dx AND dy both have a preference
    tdir = the diagonal combining them        # 45, 135, 215 or 315
    IF tdir IS NOT turnaround AND sv_step_direction(actor, tdir, dist)  RETURN

  # 2. Try the two axial directions. Swap their order randomly, or
  #    deterministically when the vertical separation dominates.
  IF (a random bit IS SET) OR |deltay| > |deltax|
    swap dx and dy
  IF dx has a preference AND is not turnaround AND stepping that way works  RETURN
  IF dy has a preference AND is not turnaround AND stepping that way works  RETURN

  # 3. Try continuing in the previous direction.
  IF olddir has a preference AND stepping that way works  RETURN

  # 4. Try all eight compass points, in a randomly chosen rotation order,
  #    skipping the turnaround.
  IF a random bit IS SET
    FOR tdir FROM 0 TO 315 STEP 45   ...try each
  ELSE
    FOR tdir FROM 315 DOWN TO 0 STEP 45   ...try each

  # 5. Only now, reverse.
  IF turnaround has a preference AND stepping that way works  RETURN

  # Nothing worked.
  actor.ideal_yaw = olddir
  IF NOT sv_check_bottom(actor)  sv_fix_check_bottom(actor)
```

**Invariants** — the whole of the game's monster navigation is this ordered preference list,
and every element of it earns its place.

**The 10-unit dead band** per axis means a monster nearly aligned with its target on one axis
treats that axis as having no preference, so it moves straight along the other rather than
weaving.

**The turnaround is tried last, always.** Reversing is the worst outcome — it makes a monster
appear to give up — so every other direction, including all eight compass points, is attempted
first.

**Two independent random bits** decide the axial order and the rotation direction of the
eight-way sweep. That is what makes two monsters in the same corridor diverge instead of both
failing identically, and it is why monster movement in this game is not reproducible frame for
frame.

**The axial order also depends on which separation is larger**, which biases the monster toward
closing the longer gap first — a crude but effective approximation of moving toward the target.

The previous direction is tried *after* the two axial ones, so a monster commits to a heading
only weakly; it re-evaluates toward its target every time it is blocked.

**Notes** — the diagonal computation has a typographical error: three of the four diagonals are
45, 135 and 315, and the fourth is written as **215** rather than 225. So a monster whose
target is behind and to its left tries a heading 10 degrees off the true diagonal. It still
works, because the step is then vetoed or accepted on its merits and the monster re-evaluates,
but the heading is wrong. A rebuild must decide: reproducing it preserves the original's exact
movement, and fixing it changes monster paths in every published map. The recipe recommends
reproducing it and noting it, because monster paths are part of the game's difficulty tuning.

The array holding the two axis preferences is declared with three elements and indexed from 1,
so its first element is unused. Incidental.

The headings are compared for equality against the turnaround, which works only because every
heading in play is an exact multiple of 45 produced by the same snapping. Another equality
comparison made safe by quantization, as in [`pr_cmds.c`](pr_cmds.c.md#pf_changeyaw-changeyaw-49).

## `SV_CloseEnough`

**Contract** — takes an entity, a goal and a distance; reports whether the two bounding boxes
are within that distance of each other on every axis.

**Invariants** — an axis-aligned proximity test, not a radial one, so a diagonal separation
counts as closer than it is.

## `SV_MoveToGoal`

**Contract** — a host function ([`pr_cmds.c`](pr_cmds.c.md)) taking a distance; moves the
current entity one step toward its goal entity. Returns zero without moving if the entity is
neither supported, flying nor swimming. Returns immediately without moving if the goal is
already within reach. Otherwise continues in its current heading, or picks a new one — with a
one-in-four chance of re-picking even when the current heading would work.

```text
FUNCTION sv_move_to_goal()
  ent  = the current entity
  goal = ent.goalentity
  dist = argument 0

  IF ent has NONE OF { on-ground, flying, swimming }  RETURN 0

  IF ent.enemy IS NOT the world AND sv_close_enough(ent, goal, dist)
    RETURN                             # already there; note: returns NOTHING

  IF (a random value modulo 4) == 1 OR NOT sv_step_direction(ent, ent.ideal_yaw, dist)
    sv_new_chase_dir(ent, goal, dist)
```

**Invariants** — the **one-in-four random re-pick** is the interesting decision. Even when
continuing straight ahead would work, a quarter of the time the monster re-evaluates its
direction anyway. That is what makes monster approach paths look searching rather than
mechanical, and removing it makes monsters move in dead-straight lines.

The proximity test is gated on the entity having an *enemy*, while the distance is measured to
its *goal*. Those are usually the same entity but need not be — a monster can have a goal that
is a path corner while its enemy is the player — and in that case the test compares the wrong
pair. Reproduce it; published monster behaviour depends on it.

The early return sets no return value, so the game logic sees whatever was in the return slot.
Published logic ignores the result of this call.
