# QW/server/sv_ents.c

> Builds each client's view of the world as a difference from the last snapshot that client confirmed receiving, with players sent separately, nails packed six bytes each, and visibility computed from a slightly enlarged set of leaves.

**Needs** — [`qwsvdef.h`](qwsvdef.h.md) · [`server.h`](server.h.md) · [`protocol.h`](../client/protocol.h.md) · [`model.c`](model.c.md) · [`common.h`](../client/common.h.md) · [`net_chan.c`](../client/net_chan.c.md)
**Used by** — [`sv_send.c`](sv_send.c.md) calls it once per client per frame
**Tier floor** — none

## Purpose

The second of QuakeWorld's three defining ideas (the others are in
[`net_chan.c`](../client/net_chan.c.md) and [`pmove.c`](../client/pmove.c.md)). The original engine sends each entity's
state as a difference from a *fixed* baseline written at map load
([`sv_main.c`](../../WinQuake/sv_main.c.md)); with the baseline unchanging, a moving entity costs the same bytes forever,
and the server must send reliably because a lost update would be lost permanently.

This file changes the reference point: **each update is a difference from the last snapshot the client acknowledged.** A
stationary entity then costs nothing, a moving one costs only what changed, and no update need be reliable at all, because
the next snapshot is a difference from whatever the client actually has. The acknowledgement comes free in the channel
header.

The cost is that the server keeps the last several snapshots per client, and that is the only price.

## State

```text
VARIABLE fatpvs : bitset over the map's leaves ;  fatbytes : int
CONSTANT max_nails = 32
VARIABLE nails : entity[32] ;  numnails : int

# per client, in the client record (see server.h):
RECORD ClientFrame                       # a ring of the last several, indexed by sequence
  entities : packet_entities             # the snapshot sent in that packet
  senttime : time ;  ping_time : real
  areabits : ...
CONSTANT update_backup / update_mask     # how many snapshots are remembered
CONSTANT max_packet_entities             # the per-snapshot entity limit
```

**Invariants** —

- **Snapshots are remembered in a ring indexed by the outgoing packet sequence**, so "the snapshot the client
  acknowledged" is found by one masked index with no search. The ring size bounds how far behind a client may be: a client
  whose last acknowledgement is older than the ring must be sent a full snapshot instead
  ([`sv_send.c`](sv_send.c.md) decides this).
- An entity's identifier must fit in **nine bits**, because the remaining seven bits of the leading word are field flags. So
  a map may hold at most 512 entities in a snapshot. That limit is in the protocol, not in the server, and it is a hard
  contract with the client.

## `SV_FatPVS`, `SV_AddToFatPVS`

**Contract** — computes the union of the visibility sets of every leaf within a short distance of a point, as a bitset over
the map's leaves.

```text
FUNCTION add_to_fat_pvs(org, node)
  LOOP
    IF node is a leaf
      IF it is not solid  OR the leaf's visibility set INTO the accumulator
      RETURN
    d = distance from org to the node's plane
    IF d >  8  descend the front child
    IF d < -8  descend the back child
    ELSE       recurse into the front child and continue into the back one
```

**Invariants** — **the point is treated as a small sphere, not a point**, and every leaf it could be in contributes. The
reason is stated in the source and is worth keeping: the client's view bobs as the player walks, and a view that crosses a
leaf boundary — most visibly a water surface — would otherwise pop entities in and out. The enlargement is a few units and
it removes a whole class of visual glitch.

This is the same recursion as the visibility lookup in [`world.c`](world.c.md), with the "straddles the plane, take both
sides" case that a point query does not need.

## `SV_WriteEntitiesToClient`

**Contract** — builds and writes one client's snapshot: find the client's visible set from its eye position, write every
visible player, then collect every visible non-player entity with a model into the snapshot — diverting nails to their own
list — and emit the snapshot as a difference from the client's acknowledged one, followed by the nail message.

```text
FUNCTION write_entities_to_client(client, msg)
  frame = client's snapshot ring slot for the packet being built
  pvs = fat_pvs(client's eye position)
  write_players_to_client(client, pvs, msg)
  frame.entities.count = 0 ;  numnails = 0
  FOR EACH entity after the player slots
    SKIP if it has no model
    SKIP if none of its leaves is in pvs
    IF it is a nail  add it to the nail list ;  CONTINUE
    IF the snapshot is full  CONTINUE                # silently dropped
    copy its identifier, origin, angles, model, frame, colour map,
      skin and effects into the snapshot
  emit_packet_entities(client, frame.entities, msg)
  emit_nail_update(msg)
```

**Invariants** —

- **An entity's visibility is decided by whether any leaf it touches is visible**, using the leaf list the world stored when
  the entity was linked ([`world.c`](world.c.md)). That is why linking records leaf numbers at all.
- The snapshot is a **fixed-size array and overflow is silent**: excess entities simply do not appear. A busy scene therefore
  loses entities rather than the frame, which is the right failure but it is not reported. A rebuild should at least count
  it.
- The client's own entity and the other players are **not** in this list; players are written separately below. That split
  exists because a player carries different fields from an ordinary entity and is worth its own encoding.
- The eye position, not the origin, drives visibility, because that is where the player is looking from.

## `SV_EmitPacketEntities`

**Contract** — writes the snapshot as a difference from the client's acknowledged one, by walking both lists in identifier
order and emitting one of three things per identifier: a delta, a full state from the entity's baseline, or a removal. Marks
the message as a delta against a named sequence, or as absolute when there is nothing to delta from.

```text
FUNCTION emit_packet_entities(client, new, msg)
  IF the client has acknowledged a snapshot
    old = that snapshot ;  write the delta header with its sequence number
  ELSE
    old = empty ;  write the absolute header
  WHILE either list has entries left
    newnum = next identifier in new, or infinity
    oldnum = next identifier in old, or infinity
    IF newnum == oldnum   write_delta(old entry, new entry, forced = no)   ; advance both
    IF newnum <  oldnum   write_delta(baseline,  new entry, forced = yes)  ; advance new
    IF newnum >  oldnum   write (oldnum WITH the remove flag)              ; advance old
  write a zero identifier to end the list
```

**Invariants** —

- **Both lists are kept sorted by identifier**, which is what makes the comparison a single merge pass with no lookups. The
  collection loop above walks entities in identifier order, so the sort is free — and a rebuild must preserve that, because
  an unsorted list silently corrupts the merge.
- **Three cases, and the middle one is the subtle one**: an entity present now and absent from the reference is sent as a
  difference from its *baseline*, not absolutely, because the baseline already carries its model, skin and effects. So a
  newly visible entity still costs only what differs from how the map declared it.
- **The forced flag matters.** A delta with no changed fields emits nothing at all; but a new entity that happens to match
  its baseline exactly must still be announced, or the client never creates it. Hence the force.
- **A removal is a bare identifier with a flag**, one word. Removals are the reason the reference must be a snapshot the
  client actually has: deltaing against a guess would leave entities on the client forever.
- The whole message is **unreliable**, and correctness comes from the reference being acknowledged rather than from delivery.
  That is the design's central claim and it is worth stating plainly: *a delta stream is reliable without retransmission if
  and only if the reference is something the receiver has confirmed.*

## `SV_WriteDelta`

**Contract** — writes one entity's change: a leading word holding the identifier in its low bits and the more-significant
field flags in its high bits, optionally a second flag byte, then each changed field in a fixed order.

```text
FUNCTION write_delta(from, to, msg, force)
  bits = 0
  FOR EACH axis  IF the origin moved by more than 0.1  set that origin bit
  FOR EACH axis  IF the angle differs at all           set that angle bit
  IF the colour map, skin, frame, effects or model differ  set their bits
  IF any of the low nine flag bits is set  set the more-bits flag
  IF the entity is solid  set the solid flag
  FAIL if the identifier is zero or 512 or more
  IF no bits are set AND NOT force  emit nothing
  write (identifier OR the high flag bits) as one word
  IF more-bits  write the low flag byte
  write each set field, in the fixed order: model, frame, colour map, skin,
    effects, then origin and angle interleaved per axis
```

**Invariants** —

- **The identifier and the first seven flags share one 16-bit word**, and a second byte is spent only when more flags are
  needed. So the common case — an entity that only moved — costs three or five bytes total. That packing is where the
  bandwidth saving actually comes from, and it is worth reproducing exactly if wire compatibility matters.
- **Position uses a dead band and angles do not.** A movement under a tenth of a unit is treated as no movement, because the
  wire resolution is an eighth of a unit and a jittering entity would otherwise cost bytes forever. Angles are compared
  exactly because they are already quantized to a byte by the time they are compared.
- The **removal flag and the identifier share the same word**, which is why the code asserts that a normal update never has
  that flag set — an entity identifier large enough to collide with it would corrupt the stream.
- Origin and angle are **interleaved per axis** in the output, not grouped. There is no reason for it beyond the order the
  code was written in, but it is part of the wire format and must be matched.
- The field order and the flag assignments are the protocol ([`protocol.h`](../client/protocol.h.md)); nothing here may be
  reordered independently of the client.

## `SV_WritePlayersToClient`

**Contract** — writes one message per visible connected player: a flag word, the origin and animation frame always, then
optionally the time since that player's last command was run, the command itself, the velocity per axis, the model, skin,
effects and weapon animation frame.

```text
FUNCTION write_players_to_client(client, self, pvs, msg)
  FOR EACH connected player p
    IF p is not self AND not the one being spectated
      SKIP if p is a spectator
      SKIP if none of p's leaves is in pvs
    flags = time-since-command + command
    IF p's model is not the standard player model      set the model flag
    FOR EACH axis  IF the velocity is nonzero          set that velocity flag
    IF p has effects, a non-default skin, is dead, or is crouched/gibbed
                                                       set the matching flags
    IF p is a spectator      keep ONLY the velocity flags
    ELSE IF p is self        CLEAR time-since-command and command
                             and set the weapon-frame flag if a weapon is animating
    write the player message: identifier, flags, origin, animation frame,
      then each flagged field in order
```

**Invariants** —

- **A player's own entity is sent without its command or timing**, because the client already knows what it sent and is
  predicting from it ([`cl_pred.c`](../client/cl_pred.c.md)). Sending it back would be redundant and would fight the
  prediction.
- **The weapon animation frame is sent only for yourself or for whoever you are spectating**, because it is only ever drawn
  in first person.
- **The time since that player's last command was executed is sent**, capped to a byte of milliseconds. The client uses it to
  extrapolate that player forward to the present, which is how other players move smoothly at a packet rate below the frame
  rate ([`cl_ents.c`](../client/cl_ents.c.md)). This one byte is what makes other players look fluid.
- **The command that produced the state is sent**, delta-encoded against an empty command, so the client knows the player's
  aim and movement intent and can extrapolate along it rather than in a straight line. **Buttons and impulses are zeroed
  before sending**, deliberately: knowing when a distant opponent pressed fire is information the client has no business
  having. A rebuild should keep that instinct — *send what is needed to draw, not what is known.*
- **A dead player's view angles are flattened**, so a corpse does not appear to look around. A detail, and visible.
- **A spectator is reduced to origin and velocity only**, and is invisible to non-spectators — which is why the visibility
  test is skipped for the player being followed.
- Velocity is sent as three 16-bit values, used for extrapolation, not for physics.

## `SV_AddNailUpdate`, `SV_EmitNailUpdate`

**Contract** — recognize a nail by its model index and divert it to a small list; then write the whole list as one message of
six bytes each, holding position and two angles in packed bit fields.

```text
FUNCTION emit_nail_update(msg)
  IF the list is empty  RETURN
  write the nails message kind, then the count
  FOR EACH nail
    x = (origin.x + 4096) / 2            # 12 bits, 2-unit resolution
    y, z likewise
    pitch = (16 * angle.pitch / 360) AND 15     # 4 bits
    yaw   = (256 * angle.yaw   / 360) AND 255   # 8 bits
    pack x, y, z, pitch, yaw into 48 bits and write six bytes
```

**Invariants** —

- **This exists because one weapon fills the air with projectiles.** Twenty nails as ordinary entities would be the whole
  packet; at six bytes each they are 120 bytes. The optimization is content-specific — it knows which models are nails — and
  it is the clearest example in the engine of *the protocol being shaped by what the game actually does*.
- The cost is **two-unit position resolution and coarse angles**, acceptable because a nail is small, fast and never looked
  at closely. A rebuild should recognize the general technique: for a class of objects that is numerous, short-lived and
  imprecise, a special encoding beats a general one by an order of magnitude.
- The position offset assumes coordinates within a bounded range, which is the same map-size assumption the coordinate
  encoding makes ([`common.c`](../client/common.c.md)).
- Nails are **not delta-compressed and not remembered in the snapshot**, so they cost their full size every frame and vanish
  with no removal message. The list is capped, and excess nails are dropped silently — but note that a nail beyond the cap
  still reports itself as handled, so it is *not* added to the ordinary entity list either. That is deliberate: better a
  missing nail than a nail that costs thirty bytes.

**Notes** — the three ideas in this file — deltas against an acknowledged reference, a separate and reduced player encoding,
and a special case for the numerous object — are independent and a rebuild can adopt them one at a time. The first is the one
that matters.
