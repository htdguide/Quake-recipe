# WinQuake/cl_main.c

> The client's frame: interpolates every entity between the last two server updates, attaches the visual effects an entity's model and state ask for, and builds the visible list.

**Needs** — [`client.h`](client.h.md) · [`quakedef.h`](quakedef.h.md) · [`render.h`](render.h.md) · [`model.h`](model.h.md) · [`net.h`](net.h.md) · [`protocol.h`](protocol.h.md) · [`sound.h`](sound.h.md) · [`screen.h`](screen.h.md) · [`console.h`](console.h.md) · [`input.h`](input.h.md) · [`view.h`](view.h.md) · [`cvar.h`](cvar.h.md) · [`cmd.h`](cmd.h.md) · [`zone.h`](zone.h.md) · [`server.h`](server.h.md) (to detect a local server)
**Used by** — [`host.c`](host.c.md) calls the frame and the send; [`host_cmd.c`](host_cmd.c.md) connects and disconnects
**Tier floor** — none

## Purpose

Three things, and the first is the one that makes this game feel smooth.

**Interpolation.** The server ticks at up to 72 Hz and sends at that rate; the client draws at whatever rate it
manages. Every entity's drawn position is a blend between the last two received positions, at a fraction
derived from the client's clock. The rules for choosing that fraction — and for *not* interpolating — are the
substance of the file.

**Effect attachment.** An entity's model flags and its state's effect bits are turned into dynamic lights and
particle trails here, on the client, per frame. So the server never sends a light or a particle.

**The handshake replies.** Four stages, each answered with a console command sent to the server.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## `CL_LerpPoint`

**Contract** — returns the interpolation fraction between the last two server messages, and sets the client's
clock. Returns 1 — no interpolation — when the two timestamps are equal, when interpolation is disabled, during
a timed benchmark, or when a local server is running.

```text
FUNCTION cl_lerp_point() -> real
  f = cl.mtime[0] - cl.mtime[1]            # the interval between the last two
  IF f == 0 OR interpolation is disabled OR timing a demo OR a server is LOCAL
    cl.time = cl.mtime[0] ;  RETURN 1      # snap to the newest
  IF f > 0.1
    # A dropped packet, or the start of a demo: pretend the interval was 0.1.
    cl.mtime[1] = cl.mtime[0] - 0.1 ;  f = 0.1
  frac = (cl.time - cl.mtime[1]) / f
  IF frac < 0
    IF frac < -0.01  cl.time = cl.mtime[1]     # the clock fell behind: snap
    frac = 0
  ELSE IF frac > 1
    IF frac > 1.01   cl.time = cl.mtime[0]     # the clock ran ahead: snap
    frac = 1
  RETURN frac
```

**Invariants** — four decisions.

**A local server disables interpolation entirely.** With the server in the same process there is no latency to
hide and the newest update is always current, so blending would only add a frame of lag. That is why
single-player movement feels tighter than networked movement in this engine, and it is a deliberate choice
rather than an accident.

**An interval longer than a tenth of a second is treated as a tenth.** A dropped packet would otherwise make
the client interpolate slowly across a long gap, which reads as everything moving in slow motion. Clamping makes
it catch up instead.

**The clock is snapped when it drifts more than one percent outside the interval.** The client's clock advances
by its own frame time and the server's advances by its tick, so they drift; the snap is the correction, and the
one-percent tolerance keeps it from firing on ordinary jitter.

**The fraction is clamped without snapping inside the tolerance band**, so small drift is absorbed by clamping
rather than by a visible jump.

## `CL_RelinkEntities`

**Contract** — once per frame: computes the interpolation fraction, interpolates the player's own velocity and
(during demo playback) view angles, then for each entity — drops it if it was not in the last message,
interpolates or snaps its position and angles, applies model-driven rotation, spawns whatever lights and
particles its effects and model flags ask for, and appends it to the visible list unless it is the player's own.

```text
FUNCTION cl_relink_entities()
  frac = cl_lerp_point()
  cl_numvisedicts = 0
  cl.velocity = blend(cl.mvelocity[1], cl.mvelocity[0], frac)
  IF playing back a demo
    cl.viewangles = blend of the two received angle sets, taking the SHORT way
                    round each axis
  bobjrotate = anglemod(100 * cl.time)      # a shared spin phase

  FOR EACH entity i FROM 1 upward
    IF it has no model
      IF it was force-linked  remove its fragments      # it just became empty
      CONTINUE
    IF ent.msgtime != cl.mtime[0]
      ent.model = nothing ;  CONTINUE        # not in the last packet: gone

    oldorg = ent.origin
    IF ent.forcelink
      ent.origin = ent.msg_origins[0] ;  ent.angles = ent.msg_angles[0]
    ELSE
      f = frac
      FOR EACH axis j
        delta[j] = ent.msg_origins[0][j] - ent.msg_origins[1][j]
        IF |delta[j]| > 100  f = 1           # assume a TELEPORT, not motion
      FOR EACH axis j
        ent.origin[j] = ent.msg_origins[1][j] + f*delta[j]
        d = the angle difference, taken the SHORT way round
        ent.angles[j] = ent.msg_angles[1][j] + f*d

    IF the MODEL has the rotate flag  ent.angles[YAW] = bobjrotate

    # --- effects from the entity's STATE ---
    IF ent.effects HAS bright-field   spawn the sparkle field
    IF ent.effects HAS muzzle-flash
      light at the entity's origin raised 16, pushed 18 units FORWARD along its
      facing, radius 200 plus a small random, minimum 32, lasting 0.1 s
    IF ent.effects HAS bright-light   light of radius 400, lasting 0.001 s
    IF ent.effects HAS dim-light      light of radius 200, lasting 0.001 s

    # --- effects from the MODEL's own flags ---
    IF the model HAS gib        trail of kind 2 FROM oldorg TO the new origin
    ELSE IF zombie-gib          trail of kind 4
    ELSE IF tracer              trail of kind 3
    ELSE IF tracer2             trail of kind 5
    ELSE IF rocket              trail of kind 0, AND a light of radius 200
    ELSE IF grenade             trail of kind 1
    ELSE IF tracer3             trail of kind 6

    ent.forcelink = false
    IF i IS the view entity AND the chase camera is off  CONTINUE
    append ent TO the visible list, if there is room
```

**Invariants** — seven load-bearing decisions.

**An entity absent from the last message has its model cleared** rather than being removed, so its slot survives
and a later update revives it. That is why an entity leaving the visible set *freezes* rather than vanishing —
it is not in the message, so it is dropped from the visible list, and when it comes back it resumes from its old
position.

**A position delta over 100 units on any axis is treated as a teleport** and the interpolation is skipped for
*all three axes*. That is the client-side counterpart of the protocol's teleport flag
([`protocol.h`](protocol.h.md)) and it catches teleports the server did not flag.

**Angle interpolation takes the short way round**, correcting a difference outside ±180 by a full turn. Without
it an entity turning through north spins the wrong way through 359 degrees.

**A model flagged to rotate gets a *shared* phase**, computed once per frame from the clock. So every item in a
level spins in unison, which is deliberate: they are meant to look like a set.

**A muzzle flash is positioned 18 units in front of the entity and 16 units up**, derived from the entity's own
angles — which is why the flash appears at the weapon's muzzle rather than at the entity's feet.

**A trail is drawn from the entity's *previous* position to its new one**, which is why the old position is
saved before interpolation. An entity that teleported therefore draws a trail across the map — unless the
teleport was flagged, in which case the previous position was already the new one.

**Lights from the three continuous effects last one millisecond**, which means they exist for exactly the frame
that spawns them. They are re-spawned every frame while the effect bit is set, and the light allocator's key
([`CL_AllocDlight`](#cl_allocdlight)) makes each replace its predecessor rather than accumulating.

## `CL_AllocDlight`

**Contract** — takes a key; returns a light slot, preferring one already holding that key, then any expired one,
and finally slot zero. Clears the slot and records the key.

**Invariants** — **the key is the owning entity's number**, so a rapid-firing weapon's muzzle flash replaces its
own previous light rather than consuming a new slot each frame. Without it, thirty-two slots would be exhausted
in half a second. Falling back to slot zero rather than failing means a light is always produced, at the cost of
stealing one.

## `CL_DecayLights`

**Contract** — shrinks every live light's radius by its decay rate times the frame's elapsed time, clamping at
zero.

## `CL_SignonReply`

**Contract** — sends the client's reply for the current handshake stage: ask for the signon block; then send the
player's name and colours followed by the spawn request with any level arguments; then declare readiness; then,
at the final stage, re-enable screen updates.

```text
FUNCTION cl_signon_reply()
  SELECT cls.signon
    1  send the string command "prespawn"
    2  send "name \"<the name variable>\""
       send "color <shirt> <trousers>"           # unpacked from one byte
       send "spawn <the spawn parameters>"
    3  send "begin" ;  report the remaining cache memory
    4  end the loading plaque
```

**Invariants** — the client's identity is sent as **console commands**, not as protocol messages
([`host_cmd.c`](host_cmd.c.md#host_name_f-name) receives them), which is why the name and colour commands exist at
all and why they are in the server's authorization list
([`sv_user.c`](sv_user.c.md#the-authorization-list)).

The spawn parameters are the level command's extra arguments, carried so a restart can repeat them.

Stage 4 has no reply; it only ends the loading screen, which is the moment the game becomes playable.

## `CL_EstablishConnection`, `CL_Disconnect`, `CL_Disconnect_f`, `CL_ClearState`

**Contract** — `CL_EstablishConnection` opens a connection to a named host, or fails with a host error.
`CL_Disconnect` stops sounds and demo playback, sends a disconnect, closes the connection, and — if a local
server is running — shuts it down too. `CL_ClearState` wipes the per-connection state and resets the entity,
fragment and particle pools.

**Invariants** — disconnecting **also shuts down a local server**, which is what makes a single-player
disconnect end the game rather than leave a headless server running.

## `CL_ReadFromServer`

**Contract** — advances the client's clock by the frame time, then reads and decodes every pending message,
relinks the entities, and returns. A message-level failure is a host error.

## `CL_SendCmd`

**Contract** — when fully connected, builds a movement command from the keyboard, lets the input backend add to
it, and sends it unreliably. Then, if there is anything in the reliable buffer and the channel will take it,
sends that. During demo playback, discards the reliable buffer instead.

```text
FUNCTION cl_send_cmd()
  IF not connected  RETURN
  IF fully signed on
    cl_base_move(cmd)        # from the keyboard
    in_move(cmd)             # the mouse and joystick ADD to it
    cl_send_move(cmd)        # unreliable
  IF playing back a demo  discard the reliable buffer ;  RETURN
  IF the reliable buffer is empty  RETURN
  IF the channel will not accept a message now  RETURN
  IF sending failed  FAIL WITH "lost server connection"
  clear the reliable buffer
```

**Invariants** — the **keyboard first, devices second** order is the contract
[`input.h`](input.h.md) describes: the backend modifies a command already built rather than the two being
merged.

A reliable message that cannot be sent this frame **stays in the buffer** and is retried, because the buffer is
only cleared on success. That is the client's half of the one-outstanding-message discipline
([`net.h`](net.h.md)).

## `CL_NextDemo`

**Contract** — advances the attract-mode playlist, wrapping, and queues the play command. Reports and stops if
the list is empty.

## `CL_Init`

**Contract** — allocates the reliable buffer, sets up input and temporary entities, and registers the client's
twenty variables and its commands.

## `CL_PrintEntities_f`, `SetPal`

**Contract** — a diagnostic listing every entity's model, frame, position and angles; and a disabled
screen-flashing debug aid.

**Notes** — the palette-flash aid is compiled out but is still *called* from the interpolation clamp, which is
where it was used to make clock drift visible. A rebuild should keep the idea — a visible signal when the
interpolation clock snaps — because drift is otherwise invisible and very hard to diagnose.
