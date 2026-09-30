# QW/client/cl_ents.c

> Receiving the world: apply each snapshot as a difference from the one it names, extrapolate every other player forward to the moment being drawn, unpack the nail encoding, and make the players solid so predicted movement collides with them.

**Needs** — [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) · [`protocol.h`](protocol.h.md) · [`pmove.h`](pmove.h.md) · [`cl_pred.c`](cl_pred.c.md) · [`net_chan.c`](net_chan.c.md) · [`cl_cam.c`](cl_cam.c.md)
**Used by** — [`cl_parse.c`](cl_parse.c.md) parses into it; [`cl_main.c`](cl_main.c.md) builds the frame's visible entity list from it
**Tier floor** — none

## Purpose

The client's half of [`sv_ents.c`](../server/sv_ents.c.md), and the file with no counterpart at all in the original engine — where
entity updates are applied directly to a persistent array
([`cl_parse.c`](../../WinQuake/cl_parse.c.md)). Here they are applied to a **snapshot in a ring**, because the next snapshot will
be a difference from this one and both must be kept.

Beyond that it owns two things the original never needed: **extrapolating other players forward in time**, and making them solid
for the local player's prediction.

## State

```text
# in the client state, a ring indexed by packet sequence:
RECORD Frame
  packet_entities : the snapshot as applied
  playerstate[]   : each player's state as reported, with the time it was valid at
  delta_sequence  : which snapshot this one was a difference from
  invalid         : bool
  cmd, senttime   # (see cl_pred.c)
VARIABLE cl.validsequence          # the newest usable snapshot, or none
VARIABLE cl.parsecount, parsecountmod, parsecounttime
VARIABLE predicted_players[]       # the extrapolated positions, for collision
VARIABLE cl_projectiles[]          # the unpacked nails
VARIABLE cl_dlights[]              # dynamic lights, allocated by owner
VARIABLE bitcounts[16]             # per-field byte accounting, for diagnosis
```

**Invariants** — **a snapshot is stored in the ring slot of the packet that carried it**, and it records which slot it was a
difference from. So applying a snapshot requires its reference slot still to be present, which bounds how far behind the client may
fall — the same bound the server enforces from its side ([`sv_ents.c`](../server/sv_ents.c.md)).

## `CL_ParsePacketEntities`

**Contract** — applies one snapshot: read which snapshot it deltas from (or note that it is absolute), verify that reference is
still available, then merge the arriving stream of changes against the reference into a new snapshot. Reports the new snapshot as
usable, or discards the packet if the reference is gone.

```text
FUNCTION parse_packet_entities(is_delta)
  new = the ring slot for the arriving packet ;  mark it valid
  IF is_delta
    read the reference sequence
    warn if it disagrees with what we asked for
    old = the reference slot
    IF the reference is older than the ring holds  flush_entity_packet() ;  RETURN
    record this as the newest usable snapshot
  ELSE
    old = empty
  merge:
  LOOP
    word = next identifier-and-flags word ;  IF zero  DONE
    newnum = its low bits
    WHILE the reference's next identifier < newnum
      carry that entity forward UNCHANGED into new       # unmentioned = unchanged
    IF the word carries the remove flag
      skip the reference's matching entry                # it is gone
    ELSE IF the reference has a matching entry
      parse_delta(that entry, into new, word)            # changed
    ELSE
      parse_delta(empty, into new, word)                 # newly visible
  carry any remaining reference entries forward unchanged
```

**Invariants** —

- **An entity the server did not mention is carried forward unchanged.** That is the whole of the delta scheme's decompression, and
  it means a stationary entity costs zero bytes indefinitely. The merge is a single pass over two sorted lists, mirroring the
  server's ([`sv_ents.c`](../server/sv_ents.c.md)) — and it depends on both being sorted by identifier.
- **A snapshot whose reference has expired is discarded entirely**, not partially applied, and the client stops rendering until an
  absolute snapshot arrives. Partially applying it would corrupt every subsequent delta. The client then asks for an absolute
  snapshot by omitting the reference request ([`cl_main.c`](cl_main.c.md)).
- A **disagreement between the reference the server names and the one the client asked for is a warning, not an error**, because
  the request and the reply cross in flight; the server's stated reference is authoritative.
- The parse must consume the whole stream even when discarding, or the rest of the packet is misread — which is what the flush path
  does.

## `CL_ParseDelta`

**Contract** — reads one entity's change: copy the reference state, take the identifier from the low bits of the leading word,
read a second flag byte if indicated, then read each flagged field in the fixed order.

**Invariants** — **the reference state is copied first and then overwritten field by field**, which is what makes an unmentioned
field mean "unchanged" at field granularity as well as at entity granularity.

The field order, the flag assignments and the nine-bit identifier are the protocol
([`protocol.h`](protocol.h.md)) and must match [`sv_ents.c`](../server/sv_ents.c.md) exactly. The per-field byte counting exists so
a developer can see where the bandwidth goes; it is worth having.

The solid flag is read and **ignored** — the field it would have carried was never implemented. Recorded because the flag occupies a
protocol bit that a rebuild must still skip.

## `CL_ParsePlayerinfo`

**Contract** — reads one player's state: their slot, a flag word, their position and animation frame always, then optionally the
time since their last command ran, the command itself, their velocity, model, skin, effects and weapon frame. Fields not present
are reset to defaults, not carried forward.

```text
FUNCTION parse_playerinfo()
  num = the player slot
  state = the arriving frame's slot for that player
  flags = the flag word ;  record which parse this state belongs to
  origin, animation frame                     # always present
  IF the time-since-command flag is set
    msec = one byte
    state.state_time = this packet's time - msec           # WHEN it was valid
  ELSE
    state.state_time = this packet's time
  IF the command flag is set  read the command, delta from empty
  FOR EACH axis  velocity = the field if flagged, ELSE zero
  model = the field if flagged, ELSE the standard player model
  skin, effects, weapon frame = the field if flagged, ELSE zero
  view angles = the command's angles
```

**Invariants** —

- **A player's state is absolute, not delta-compressed**, unlike entities. There are at most sixteen of them, they change every
  frame, and they are the thing the player is looking at — so nothing is saved by deltaing and a lost packet would cost more.
- **Absent fields reset to defaults rather than persisting.** That is the opposite convention from entities, and it is deliberate:
  the server omits a field precisely when it is at its default ([`sv_ents.c`](../server/sv_ents.c.md)), so "absent" *means*
  "default".
- **The state is stamped with the moment it was actually valid**, computed by subtracting the reported age from the packet's time.
  That one byte is what makes the extrapolation below correct rather than approximate: without it every player would be
  extrapolated from the packet's arrival, which is later than when their command ran.
- The view angles come from the command, which is how a distant player's aim is known.

## `CL_SetUpPlayerPrediction`, `CL_LinkPlayers`

**Contract** — for each player present in the newest snapshot, compute where they are at the moment being drawn: the local player
from its own prediction, and every other player by **running the movement model forward from their reported state using their
reported command**. Then build their visible entities, their held items' models, and the light their effects cast.

```text
FUNCTION set_up_player_prediction(do_predict)
  playertime = now - measured latency + a small lead, clamped to now
  FOR EACH player in the newest snapshot
    IF absent this frame OR it has no model  mark inactive ;  CONTINUE
    IF it is the local player
      position = our own latest predicted position (cl_pred.c)
    ELSE
      msec = HALF the interval from that player's state time to playertime
      IF msec <= 0 OR prediction of others is disabled
        position = the reported position
      ELSE
        clamp msec to a byte ;  set it as the command's duration
        run the movement model once from the reported state with that command
        position = the result
```

**Invariants** —

- **Other players are extrapolated by running the same movement model on their last known command.** So a player who was strafing
  keeps strafing, decelerating under the same friction, colliding with the same walls — rather than sliding linearly along their
  velocity. That is why the command is sent at all
  ([`sv_ents.c`](../server/sv_ents.c.md)), and it is the difference between opponents that move plausibly and opponents that skate
  through walls.
- **Only half the elapsed interval is extrapolated**, and the source says why: *to minimize overruns.* Extrapolating the full
  interval puts a player slightly ahead of where the next packet says they were, and the correction is then a visible backward
  jerk. Extrapolating half puts them behind, and the correction is forward — which reads as smooth motion. **Deliberately
  under-extrapolating is the right choice and it is not obvious.**
- **The local player is not extrapolated but taken from its own prediction**, which is further ahead and authoritative for
  rendering.
- The displayed moment is the present **minus the measured latency plus a small lead**, matching
  [`cl_pred.c`](cl_pred.c.md)'s displayed time so the local player and everyone else are drawn at the same instant. If the two
  disagreed, the player would lead or lag the world.
- Extrapolation is a **setting**, because on a low-latency link it is not worth the error.

## `CL_SetSolidPlayers`, `CL_SetSolidEntities`

**Contract** — add each active player's box, and each solid entity from the snapshot, to the movement world list so the local
player's prediction collides with them.

**Invariants** —

- **The prediction must collide with other players or the local player walks through them** and is then pushed out when the server
  disagrees — a rubber-band that is worse than a small position error. This is why the extrapolated positions above are computed
  before the prediction runs ([`cl_pred.c`](cl_pred.c.md) calls this, and the ordering is stated in the source as a requirement).
- **It is a setting**, because colliding with an extrapolated position is sometimes worse than not colliding at all — a
  mispredicted opponent becomes an invisible wall.
- The list is the movement model's flat list ([`pmove.h`](pmove.h.md)) with a bounded size, so in a crowd some players are omitted.

## `CL_ParseProjectiles`, `CL_LinkProjectiles`, `CL_ClearProjectiles`

**Contract** — unpack the nail message's six-byte entries into positions and angles, and build a visible entity for each; the list
is rebuilt every frame from scratch.

**Invariants** — **nails are not part of the snapshot and are not delta-compressed**: the list is cleared and refilled every
packet, so a nail simply disappears when it stops being sent, with no removal message. That is the consequence of the special
encoding ([`sv_ents.c`](../server/sv_ents.c.md)) and it is acceptable because nails are short-lived.

The unpacking must mirror the packing bit-for-bit, including the position offset and the two-unit resolution.

## `CL_LinkPacketEntities`

**Contract** — turns the newest snapshot into the frame's visible entity list: for each entity, set its model, frame, skin and
colours, interpolate its angles from the previous snapshot where appropriate, attach its effect models and trails, and spawn the
light its effects cast.

**Invariants** —

- **Angles are interpolated between snapshots and positions are not.** A rotating item looks smooth; a moving one is drawn where
  the snapshot put it. That asymmetry is a deliberate economy — position error is corrected by the next packet anyway, and
  interpolating it would add latency — and it is visible as slight stepping on fast-moving objects.
- A **trail-emitting entity's previous position must be remembered per entity** or its trail is drawn from the wrong place; that
  memory is keyed by entity identifier, which is why the identifier is stable across snapshots.
- Effects are decoded here rather than by the server, so the client's frame rate governs their smoothness.

## `CL_AddFlagModels`

**Contract** — attaches a carried flag's model to a player, positioned and angled from the player's own animation.

**Invariants** — a game-specific effect in the engine, driven by two bits of the player's effect field. Recorded as a layering
compromise: a team-game feature hard-coded in the client because the protocol had no way to say "attach this model to that player".
**That absence is the real lesson** — a rebuild should give the protocol a general attachment, and then this file needs nothing
special.

## `CL_AllocDlight`, `CL_NewDlight`, `CL_DecayLights`

**Contract** — allocate a dynamic light, reusing the slot belonging to a given owner if it exists; create the light an entity's
effect flags call for; and shrink every light each frame, freeing the expired.

**Invariants** — **lights are keyed by owner**, so a player's muzzle flash replaces their previous one rather than accumulating.
Without the keying a continuously firing player exhausts the light pool in a second.

## `FlushEntityPacket`, `CL_EmitEntities`

**Contract** — consume and discard a snapshot whose reference is unusable, marking the frame invalid; and the per-frame entry point
that builds the whole visible list — players, entities, projectiles.

**Notes** — the three ideas worth taking from this file are: **carry unmentioned entities forward** (the decompression), **stamp
each player's state with when it was valid** (the one byte that makes extrapolation correct), and **under-extrapolate deliberately**
(so corrections read as forward motion). The third is the one a rebuilder is least likely to arrive at alone.
