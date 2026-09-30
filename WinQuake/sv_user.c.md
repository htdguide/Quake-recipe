# WinQuake/sv_user.c

> Player movement: the acceleration model that defines how this game feels to play, the command decoder, and the authorization list that decides which console commands a remote player may run.

**Needs** — [`server.h`](server.h.md) · [`world.h`](world.h.md) · [`progs.h`](progs.h.md) · [`protocol.h`](protocol.h.md) · [`common.h`](common.h.md) (the message reader) · [`cmd.h`](cmd.h.md) · [`mathlib.h`](mathlib.h.md) · [`view.h`](view.h.md) (the roll calculation, shared with the client) · [`keys.h`](keys.h.md) (the single-player pause condition) · [`net.h`](net.h.md) · [`bspfile.h`](bspfile.h.md)
**Used by** — [`host.c`](host.c.md) calls the per-frame client loop; [`sv_phys.c`](sv_phys.c.md) is reached through it
**Tier floor** — none

## Purpose

Three separable things share this file and only the first is about movement.

The **acceleration model** is the heart of it: separate ground, air and water rules, with the
air rule deliberately weakened in a way that produces the movement technique this game is
remembered for. Every constant here is a feel decision.

The **command decoder** reads a player's intent off the wire and, notably, writes the player's
view angles directly into their entity where game logic can see them.

The **authorization list** is a hard-coded set of eighteen command names a remote player is
allowed to run, and everything else is refused. It is the engine's entire access control, and
it is a denylist-shaped allowlist with a prefix-matching flaw worth understanding.

## State

```text
VARIABLE sv_player  : Edict            # the player being processed
VARIABLE cmd        : UserCmd          # their movement command, copied locally
VARIABLE onground   : bool
VARIABLE angles, origin, velocity : pointers INTO sv_player's fields
VARIABLE forward, right, up : vec3     # the player's basis, this frame
VARIABLE wishdir    : vec3             # the unit direction they want to move
VARIABLE wishspeed  : real             # how fast they want to move

VARIABLE sv_maxspeed        : Cvar = 320   # units per second; server-announced
VARIABLE sv_accelerate      : Cvar = 10
VARIABLE sv_edgefriction    : Cvar = 2     # friction multiplier near a drop
VARIABLE sv_idealpitchscale : Cvar = 0.8
```

**Invariants** — the position and velocity are reached through pointers **into the entity's
fields**, so every routine here writes the entity directly with no copy back. That is why the
order of operations within a tick is observable.

The movement numbers together are the game's feel: 320 units per second maximum, acceleration
of 10, friction of 4 ([`sv_phys.c`](sv_phys.c.md)), gravity of 800, step height 18. A rebuild
changing any of them changes how the game plays.

## `SV_SetIdealPitch`

**Contract** — takes nothing; examines the floor ahead of the player and sets the pitch angle
the automatic view centring should aim for, so that walking up stairs tilts the view up. Does
nothing unless the player is on the ground, and gives up on a wall, a drop-off, or an
inconsistent slope.

```text
FUNCTION sv_set_ideal_pitch()
  IF the player is not on the ground  RETURN
  # Sample the floor height at six points ahead, 12 units apart starting
  # 36 units out, along the player's yaw.
  FOR i FROM 0 TO 5
    top    = the player's EYE position, moved forward by (i+3)*12 units
    bottom = top, lowered BY 160
    tr = sweep a POINT FROM top TO bottom, ignoring monsters
    IF tr.allsolid       RETURN      # looking into a wall: leave it alone
    IF tr.fraction == 1  RETURN      # no floor within 160 units: a drop-off
    z[i] = the floor height at that sample

  # The six heights must form a consistent staircase.
  dir = 0 ;  steps = 0
  FOR j FROM 1 TO 5
    step = z[j] - z[j-1]
    IF |step| < 0.1  CONTINUE                    # flat: ignore this sample
    IF dir != 0 AND |step - dir| > 0.1  RETURN   # slope changed: give up
    steps = steps + 1 ;  dir = step

  IF dir == 0  the player's ideal pitch = 0 ;  RETURN     # flat ground
  IF steps < 2  RETURN                          # one step is not a staircase
  the player's ideal pitch = -dir * sv_idealpitchscale
```

**Invariants** — the samples start **36 units ahead** and are 12 apart, so the routine looks
between 36 and 96 units out — roughly one to three steps. The 160-unit downward reach is about
nine step heights.

Requiring **at least two consistent steps** is what stops the view tilting at a single kerb.
Requiring the step heights to *agree* within a tenth of a unit is what stops it tilting on
rubble. A rebuild that relaxes either makes the view lurch constantly.

The result is **negated**, because positive pitch looks down
([`mathlib.c`](mathlib.c.md#anglevectors)) and climbing should look up.

**Notes** — the ideal pitch is only a *target*; the client's own automatic centring moves
toward it ([`view.c`](view.c.md)), and a player who has looked manually recently is not
affected. So this is a comfort feature, not a camera constraint.

## `SV_UserFriction`

**Contract** — reduces the player's velocity by friction, using a higher coefficient when the
edge of their footprint is over a drop.

```text
FUNCTION sv_user_friction()
  speed = the HORIZONTAL speed
  IF speed == 0  RETURN

  # Sample 16 units ahead along the direction of travel, at foot height,
  # looking 34 units down.
  probe = origin, moved 16 units ALONG the horizontal velocity, at the
          player's bottom
  tr = sweep a POINT FROM probe DOWN 34 units, ignoring monsters
  friction = sv_friction * sv_edgefriction  IF tr found NOTHING
             sv_friction                    OTHERWISE

  # The friction acts on `control`, which is the speed but never below
  # sv_stopspeed — so slow movement decelerates at a fixed rate rather
  # than asymptotically.
  control = max(speed, sv_stopspeed)
  newspeed = max(speed - host_frametime * control * friction, 0)
  velocity = velocity SCALED BY (newspeed / speed)
```

**Invariants** — three decisions, all felt.

**Edge friction doubles the deceleration when the player's leading edge is over a drop.** That
is what makes it possible to stop at the top of a ledge instead of sliding off, and it is
sampled along the *direction of travel*, not the view direction.

**The floor of the friction control at the stop speed** is what makes a player come to a
complete halt rather than creeping. Without it, deceleration is proportional to speed and never
reaches zero.

**The scale is applied to all three components**, including the vertical one, even though the
speed was computed from the horizontal ones only. So a player walking off a ledge has their
*fall* slowed by ground friction for one tick. A rebuild reproducing the original's feel must
keep it.

## `SV_Accelerate`

**Contract** — accelerates the player toward their wished direction, on the ground.

```text
FUNCTION sv_accelerate()
  currentspeed = dot(velocity, wishdir)         # how fast we already go that way
  addspeed = wishspeed - currentspeed
  IF addspeed <= 0  RETURN                      # already at or past the target
  accelspeed = min(sv_accelerate * host_frametime * wishspeed, addspeed)
  velocity = velocity + wishdir * accelspeed
```

**Invariants** — the acceleration is capped by **how much is still needed along the wished
direction**, projected. So a player moving sideways at full speed who turns can still
accelerate, because their speed *along the new direction* is low. That projection, rather than
a magnitude cap, is what the whole movement model rests on.

The acceleration rate is scaled by the wished speed, so walking (a lower wished speed)
accelerates more slowly than running — proportionally, reaching the target in the same time.

**Notes** — an earlier formulation is present and disabled, which accelerated toward a wished
*velocity* rather than along a wished *direction*. The difference is exactly the projection
above, and the disabled version does not admit the technique described below.

## `SV_AirAccelerate`

**Contract** — accelerates the player toward their wished direction while airborne, with the
wished speed clamped to 30 units per second regardless of what they asked for.

```text
FUNCTION sv_air_accelerate(wishveloc)
  wishspd = min(length(wishveloc), 30)          # THE CLAMP: at most 30
  normalize wishveloc
  currentspeed = dot(velocity, wishveloc)
  addspeed = wishspd - currentspeed
  IF addspeed <= 0  RETURN
  accelspeed = min(sv_accelerate * wishspeed * host_frametime, addspeed)
  velocity = velocity + wishveloc * accelspeed
```

**Invariants** — this is the most consequential twenty lines in the game.

The clamp limits how fast you may *newly* move in the wished direction while airborne to 30
units per second. But the **projection is onto the wished direction**, and the wished direction
is where the player is *looking and strafing*, not where they are travelling. So a player
moving forward at 400 units per second who turns slightly and holds strafe has almost no
existing speed along the new wished direction — the projection is small — and so the full 30 is
granted, *added to* their existing velocity rather than replacing it.

Repeating that every tick adds speed without bound. That is strafe jumping, and it is an
emergent consequence of a clamp on the projected component rather than on the total. It was not
designed; it is the single most famous accident in first-person-shooter movement, and every
subsequent engine in this lineage has had to decide whether to keep it.

A rebuild **must decide deliberately**. Clamping the resulting total speed removes the
technique and changes how every published map plays, because map authors used it. Keeping the
code as written keeps it.

The acceleration budget uses the *unclamped* wished speed as its scale while the target uses the
clamped one, which the source's own commented-out line shows was changed at some point. The
effect is that acceleration in air is as brisk as on the ground even though the target is tiny —
so the 30 units are granted in a fraction of a tick and the technique is reliable rather than
marginal.

## `DropPunchAngle`

**Contract** — reduces the magnitude of the player's view kick by 10 degrees per second, toward
zero.

**Invariants** — the decay is on the kick's *length*, not per axis, so the kick's direction is
preserved as it fades.

## `SV_WaterMove`

**Contract** — computes the player's movement while submerged: a wished velocity from their full
three-dimensional view basis, a sinking drift when no keys are held, a speed limit at 70 percent
of the maximum, then friction and acceleration on the full velocity including the vertical.

```text
FUNCTION sv_water_move()
  forward, right, up = the basis OF the player's VIEW angles
  wishvel = forward*cmd.forwardmove + right*cmd.sidemove
  IF no movement keys at all
    wishvel[2] = wishvel[2] - 60                 # sink slowly
  ELSE
    wishvel[2] = wishvel[2] + cmd.upmove

  wishspeed = length(wishvel)
  IF wishspeed > sv_maxspeed  scale wishvel down TO sv_maxspeed
  wishspeed = wishspeed * 0.7                    # swimming is 70% of running

  # Friction on the FULL velocity, proportional to speed — no stop-speed floor.
  speed = length(velocity)
  IF speed != 0
    newspeed = max(speed - host_frametime * speed * sv_friction, 0)
    velocity = velocity SCALED BY (newspeed / speed)
  ELSE
    newspeed = 0

  IF wishspeed == 0  RETURN
  addspeed = wishspeed - newspeed                # NOT projected: total speed
  IF addspeed <= 0  RETURN
  normalize wishvel
  accelspeed = min(sv_accelerate * wishspeed * host_frametime, addspeed)
  velocity = velocity + wishvel * accelspeed
```

**Invariants** — swimming uses the **view** basis including pitch, which is why a player swims
in the direction they look; ground movement uses the entity's angles with pitch removed.

The acceleration compares against the **total** speed, not the projected component. So the
strafe-jumping accident does not exist underwater: swimming is speed-capped for real.

Friction underwater has **no stop-speed floor**, so a player drifts to a halt asymptotically
rather than stopping — which is what makes water feel like water.

The **60-unit sinking drift** applies only when no key at all is held, including the swim-up
key, so treading water requires holding something.

## `SV_WaterJump`

**Contract** — while the water-jump state is active, forces the player's horizontal velocity to
a stored direction, and clears the state when its timer expires or the player leaves the water.

```text
FUNCTION sv_water_jump()
  IF sv.time > the player's teleport_time OR the player is not in water
    clear the water-jump flag ;  the player's teleport_time = 0
  the player's horizontal velocity = the player's stored movedir
```

**Invariants** — the state overrides player input entirely: while it is set, the player's
movement command is ignored and they travel in the stored direction. Gravity is also suspended
for them ([`sv_phys.c`](sv_phys.c.md#sv_physics_client)). That combination is what makes
climbing out of water a single smooth motion rather than a series of failed jumps.

Two fields are reused for unrelated purposes here — the teleport timer as the state's deadline,
and the movement direction as the stored velocity. Both belong to game logic, which sets them
when it decides a water jump should begin.

The clearing test runs *before* the velocity is forced, so the final tick of a water jump still
forces the velocity after clearing the flag.

## `SV_AirMove`

**Contract** — computes the player's movement out of water: a wished velocity from their
horizontal basis, then one of three rules by movement type and ground contact.

```text
FUNCTION sv_air_move()
  forward, right, up = the basis OF the player's ENTITY angles
                       # note: entity angles, whose pitch is one third of the
                       # view's and whose roll is the lean — see below
  fmove = cmd.forwardmove ;  smove = cmd.sidemove
  IF sv.time < the player's teleport_time AND fmove < 0
    fmove = 0                         # do not let them back into a teleporter

  wishvel = forward*fmove + right*smove
  wishvel[2] = cmd.upmove IF the player's movetype IS NOT walk ELSE 0

  wishdir = normalize(wishvel) ;  wishspeed = its length
  IF wishspeed > sv_maxspeed  scale wishvel down ;  wishspeed = sv_maxspeed

  IF movetype IS noclip   velocity = wishvel            # direct control
  ELSE IF onground        sv_user_friction() ;  sv_accelerate()
  ELSE                    sv_air_accelerate(wishvel)
```

**Invariants** — the basis comes from the **entity's** angles, not the view angles. The
entity's pitch is one third of the view's and its roll is the velocity-derived lean
([`SV_ClientThink`](#sv_clientthink)), so the movement basis is not quite the look direction.
The vertical component is then forced to zero for a walking player anyway, so the pitch
difference only matters for flying and noclipping players — where it is a bug that makes them
move at a third of the intended angle.

**The vertical component is zeroed for walking players**, which is why looking up and pressing
forward moves you forward rather than into the air.

The **teleporter guard** is labelled a hack in the source and prevents a player from backing
into the teleporter they just used, which would loop them.

The wished direction is normalized **before** the speed clamp, so `wishdir` is a unit vector
and `wishspeed` carries the magnitude — which is the form both accelerators expect.

## `SV_ClientThink`

**Contract** — runs one player's movement for one tick. Copies their command locally, decays
their view kick, derives their entity angles from their view angles, then dispatches to the
water-jump, water or air rule. Returns immediately for a player with no physics, and — after
decaying the kick — for a dead one.

```text
FUNCTION sv_client_think()
  IF the player's movetype IS none  RETURN
  onground = the player's on-ground flag
  origin, velocity = pointers into the player's fields
  drop_punch_angle()
  IF the player's health <= 0  RETURN            # dead players do not move

  cmd = the client's most recent movement command
  v_angle = the player's view angles + their punch angle
  the player's angles[ROLL] = the velocity-derived lean, TIMES 4
  IF NOT the player's fixangle
    the player's angles[PITCH] = -v_angle[PITCH] / 3
    the player's angles[YAW]   =  v_angle[YAW]

  IF the player has the water-jump flag  sv_water_jump() ;  RETURN
  IF the player's waterlevel >= 2 AND movetype IS NOT noclip
    sv_water_move() ;  RETURN
  sv_air_move()
```

**Invariants** — the **entity's pitch is one third of the view's, and negated.** That is the
game's signature: a player model leans a third as far as its owner is looking, so other players
can read where you are aiming without the model bending double. The negation is the pitch sign
convention.

The **roll is four times the velocity-derived lean**, computed by a routine shared with the
client ([`view.c`](view.c.md#v_calcroll)) so that both ends agree. That shared routine is the
only code in the engine used by both the server's simulation and the client's rendering.

The fixangle flag suppresses the pitch and yaw derivation, so game logic that has just
teleported a player can set their angles absolutely without this overwriting them.

A **dead player's kick still decays** but they do not move — the early return is placed after
the decay deliberately, so a corpse's view settles.

The command is copied out of the client record into a file-level variable, so the movement
routines take no arguments.

## `SV_ReadClientMove`

**Contract** — decodes one movement message: a timestamp used to compute the round trip, three
view angles written straight into the player's entity, three movement axes, a button bit set,
and an optional impulse.

```text
FUNCTION sv_read_client_move(move)
  # Round trip: the client echoes the server time it was responding to.
  the client's ping ring[num_pings modulo 16] = sv.time - read a float
  num_pings = num_pings + 1

  FOR EACH axis  the player's v_angle[axis] = read an angle    # one byte each
  move.forwardmove = read a short
  move.sidemove    = read a short
  move.upmove      = read a short

  bits = read a byte
  the player's button0 = bit 0        # attack, held
  the player's button2 = bit 1        # jump, held
  i = read a byte
  IF i != 0  the player's impulse = i     # a one-shot command number
```

**Invariants** — the **view angles are written directly into the player's entity**, not into the
movement command. So game logic sees a player's aim the instant the packet arrives, before any
movement runs. That is why hitscan weapons in this game aim where the player was looking at
send time rather than at tick time.

An impulse of zero is **not** written, so a non-zero impulse persists until game logic consumes
it by zeroing the field. That is the only piece of player input with latching semantics, and it
is how weapon selection survives a tick in which the game did not run.

Button 1 is absent from the wire: only attack and jump are transmitted, in two bits of one
byte. The third button exists in the field layout and is never set.

The angles are one byte each ([`common.c`](common.c.md#msg_readcoord-msg_readangle)), so a player's aim is
quantized to about 1.4 degrees on the server. That is coarse enough to matter for long shots and
it is a fact about this protocol, corrected in
[`QW`](../QW/client/protocol.h.md).

There is **no validation** of the movement axes: a client may send any value up to a signed
short and the server will use it, clamped only by the maximum speed. A rebuild should clamp on
receipt.

## `SV_ReadClientMessage`

**Contract** — reads and executes every message a client has sent. Returns whether the client
should be kept. A network failure, a malformed read, a disconnect message, an unknown message
tag, or a command that deactivated the client all mean drop.

```text
FUNCTION sv_read_client_message() -> bool
  REPEAT
    ret = get the next message for this client
    IF ret == -1  print and RETURN false          # the connection died
    IF ret == 0   RETURN true                     # nothing pending
    begin reading
    LOOP
      IF the client is no longer active  RETURN false   # a command errored
      IF the read overran               print and RETURN false
      SELECT the next tag byte
        -1 (end of message)  fetch the next message
        nop                  nothing
        disconnect           RETURN false
        move                 sv_read_client_move(the client's command)
        stringcmd            see the authorization below
        otherwise            print and RETURN false
  WHILE ret == 1
  RETURN true
```

### The authorization list

```text
# Inside the string-command case:
allowed = 2 IF the client is privileged ELSE 0
IF the string BEGINS WITH any of:
     "status" "god" "notarget" "fly" "name" "noclip" "say" "say_team"
     "tell" "color" "kill" "pause" "spawn" "begin" "prespawn"
     "kick" "ping" "give" "ban"
  allowed = 1

IF allowed == 2   insert the string at the FRONT of the server's command queue
ELSE IF allowed == 1
  execute the string immediately, marked as coming FROM A CLIENT
ELSE
  print "<name> tried to <string>"      # and do nothing
```

**Invariants** — this is the engine's entire access control for remote players, and it has four
properties a rebuild must understand.

**It is an allowlist of nineteen prefixes**, and everything else is refused with a log line. So
a client cannot run arbitrary console commands, cannot set variables, and cannot execute
scripts.

**Matching is by prefix, case-insensitively, with the length of the *expected* name.** So
`killserver` matches `kill`, `godmode` matches `god`, and `nameless` matches `name`. Every such
collision is then resolved by the command dispatcher, which does an exact match and reports the
longer name as unknown — so the flaw is contained. But a rebuild that adds a command whose name
extends one of these nineteen grants remote access to it accidentally. Match whole words.

**A privileged client bypasses the list entirely** and has its string *inserted at the front of
the server's own command queue*, which means it runs as if typed locally, with full console
authority. The privilege flag is never set anywhere in this build
([`sv_main.c`](sv_main.c.md)), so the path is dead — but it is a remote-code-execution path
waiting for someone to set the flag, and a rebuild should delete it rather than port it.

**Allowed commands run with the source marked as coming from a client**, which is what lets each
handler apply its own further restrictions ([`cmd.h`](cmd.h.md)). Four of the nineteen —
the cheats and the administrative pair — check it and refuse.

The list includes the three connection-handshake commands, so the handshake travels through the
same path as chat.

## `SV_RunClients`

**Contract** — for each active player slot: read their messages, drop them if that failed, clear
their movement command if they have not yet entered the world, and otherwise run their movement
— unless the game is paused, or unless this is a single-player game whose console or menu is
open.

```text
FUNCTION sv_run_clients()
  FOR EACH client slot
    IF not active  CONTINUE
    host_client = this slot ;  sv_player = its entity
    IF NOT sv_read_client_message()
      sv_drop_client(crashed = false) ;  CONTINUE
    IF NOT spawned
      clear their movement command ;  CONTINUE
    IF NOT sv.paused AND (there is more than one player OR the game has focus)
      sv_client_think()
```

**Invariants** — the current client and current player are set as **globals** before each
iteration, because everything downstream reads them rather than taking arguments.

Clearing an unspawned client's command is what stops a player still loading from moving.

The last condition is the **single-player pause**: with one player, movement is suspended
whenever the console or a menu has the keyboard, without the game being formally paused. That
is why a solo game freezes when you open the console and a multiplayer game does not, and it is
the only place the server consults the client's input focus — a genuine layering violation that
exists because both live in one process.
