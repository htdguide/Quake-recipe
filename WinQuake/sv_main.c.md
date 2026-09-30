# WinQuake/sv_main.c

> The server's outward face: level spawning, the connection handshake, visibility culling with a deliberately fattened query, and the per-frame message composition — including the delta encoding that makes this protocol both cheap and wrong for the internet.

**Needs** — [`server.h`](server.h.md) · [`progs.h`](progs.h.md) · [`protocol.h`](protocol.h.md) · [`net.h`](net.h.md) · [`world.h`](world.h.md) · [`model.h`](model.h.md) (the visibility query) · [`common.h`](common.h.md) · [`quakedef.h`](quakedef.h.md) · [`client.h`](client.h.md) (the local client's state, for the dedicated test) · [`pr_exec.c`](pr_exec.c.md) · [`pr_edict.c`](pr_edict.c.md) · [`sv_phys.c`](sv_phys.c.md) · [`sv_user.c`](sv_user.c.md) · [`cvar.h`](cvar.h.md) · [`cmd.h`](cmd.h.md)
**Used by** — [`host.c`](host.c.md) drives everything here; [`host_cmd.c`](host_cmd.c.md) spawns levels and drops players; [`pr_cmds.c`](pr_cmds.c.md) uses the sound, particle and model-index helpers
**Tier floor** — none

## Purpose

Four jobs, and two of them define the generation.

**The delta encoding** compares each entity against its *spawn baseline* and sends the fields
that differ. That is cheap and stateless — the server keeps nothing per client — and it is
fundamentally wrong once packets are lost, because a client that misses an update never learns
about it. The whole of [`QW`](../QW/server/sv_ents.c.md) is the correction.

**The fattened visibility query** unions the visibility sets of every leaf within eight units of
the viewer, rather than using the viewer's own leaf. The reason is stated in the source and is
worth keeping: the client's view bobs, and a bob that crosses a leaf boundary would otherwise
make entities flicker in and out.

The remaining two jobs — level spawning and the connection handshake — are ordinary, but the
spawn sequence's *order* is load-bearing and easy to get wrong.

## State

```text
VARIABLE sv  : Server              # the per-level record
VARIABLE svs : ServerStatic        # the per-process record
VARIABLE localmodels : text[256]   # the names "*1", "*2", ... for sub-models

VARIABLE fatbytes : int            # bytes in the fattened visibility set
VARIABLE fatpvs   : bytes          # the set itself, one bit per map leaf
```

## `SV_Init`

**Contract** — registers the ten movement and aiming variables that live in
[`sv_phys.c`](sv_phys.c.md), [`sv_user.c`](sv_user.c.md) and
[`pr_cmds.c`](pr_cmds.c.md), and builds the sub-model name table.

**Invariants** — the sub-model names are the literal strings `*1` through `*255`. A map's moving
brushes are sub-models of the map file, and the protocol refers to them by these names, so a
door's model name is `*3`. The names are built once and never freed, and the leading asterisk is
what [`model.c`](model.c.md#mod_forname) uses to recognize a sub-model request.

## Event messages

### `SV_StartParticle`

**Contract** — appends a particle burst to the frame's unreliable broadcast: a position, a
direction quantized to sixteenths of a unit in a signed byte per axis, a count and a colour.
Drops the message silently if the datagram has less than sixteen bytes of room.

**Invariants** — the direction is clamped to the signed byte range *after* scaling by 16, so the
maximum expressible speed is about 8 units per second per axis. Particles asked for faster
directions are clamped, which is visible: fast particle sprays are less directional than the game
logic intended.

Dropping on a nearly-full datagram rather than overflowing is the pattern throughout this file,
and the sixteen-byte margin is a conservative guess at the largest message.

### `SV_StartSound`

**Contract** — appends a sound to the frame's unreliable broadcast. Validates the volume,
attenuation and channel, finds the sound's precache index by name, and packs the entity and
channel into one 16-bit field. Omits the volume and attenuation when they are the defaults.
Positions the sound at the **centre of the entity's bounding box**, not at its origin. Drops
silently on a nearly-full datagram, and prints and returns on an unprecached name. Out-of-range
arguments are a fatal error.

```text
FUNCTION sv_start_sound(entity, channel, sample, volume, attenuation)
  IF volume outside 0..255        FAIL WITH "volume = <n>"
  IF attenuation outside 0..4     FAIL WITH "attenuation = <n>"
  IF channel outside 0..7         FAIL WITH "channel = <n>"
  IF the datagram has under 16 bytes free  RETURN

  sound_num = the index OF sample IN sv.sound_precache        # a linear scan
  IF not found  print "not precached" ;  RETURN

  # Entity and channel share one 16-bit field: channel in the low 3 bits.
  packed = (entity's number SHIFTED LEFT 3) BITOR channel

  mask = 0
  IF volume      != 255  mask = mask BITOR the volume bit
  IF attenuation != 1.0  mask = mask BITOR the attenuation bit

  write the sound tag, mask, the present optional bytes, packed, sound_num,
        and the three coordinates OF the entity's bounding-box CENTRE
```

**Invariants** — the **bounding-box centre, not the origin**. An entity's origin is at its feet
for a player and at an arbitrary point for a brush, so sounds positioned at the origin would come
from the floor. This matters for spatialization and a rebuild that uses the origin gets audibly
wrong positions for doors and platforms.

The packing caps entities at 8191 for sound purposes, well above the 600 limit.

The attenuation is scaled by 64 into a byte, so its resolution is 1/64 and its range is the
validated 0 to 4.

## Client connection

### `SV_SendServerinfo`

**Contract** — composes the message a client receives on connecting or on a level change: a
version banner, the protocol version, the player limit, the game type, the level's title, the
model list, the sound list, the music track, which entity the client's camera follows, and the
handshake stage. Marks the client as needing a signon flush and as not yet spawned.

```text
FUNCTION sv_send_serverinfo(client)
  write a print message: "VERSION <v> SERVER (<program checksum> CRC)"
  write the serverinfo tag
  write the protocol version (a long)
  write the maximum player count (a byte)
  write the game type: deathmatch IF deathmatch AND NOT coop, ELSE cooperative
  write the world entity's message field            # the level's title
  write each model precache name FROM INDEX 1, then an empty string
  write each sound precache name FROM INDEX 1, then an empty string
  write the music track twice                       # track and loop track
  write the set-view tag and this client's entity number
  write the signon-stage tag and the value 1
  client.sendsignon = true ;  client.spawned = false
```

**Invariants** — the lists start at **index 1**, because index 0 is a placeholder (the empty
string at the start of the program's string table). So the client's own list is built with the
same offset and the indices agree.

The version banner begins with a control byte that selects the alternate glyph set
([`SYSTEM-REQUIREMENTS.md`](../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)), so the banner
renders highlighted.

The program checksum is sent so a client can notice it is running against a different game build
([`progdefs.q1`](progdefs.q1.md)).

The client is explicitly marked **not spawned**, so this message can be resent on a level change
and the client will redo the whole handshake.

### `SV_ConnectClient`

**Contract** — initializes a player slot for a new connection. Zeroes the record while preserving
the network connection and, when restoring a saved game, the carry values; names the player
"unconnected"; binds their entity; sets up their reliable buffer as overflow-tolerant; and either
preserves their carry values or asks the game for fresh ones. Then sends the server information.

```text
FUNCTION sv_connect_client(clientnum)
  client = slot clientnum ;  ent = entity (clientnum + 1)
  IF restoring a saved game  save the carry values aside
  connection = client.netconnection
  zero the whole client record
  client.netconnection = connection
  client.name = "unconnected" ;  client.active = true ;  client.spawned = false
  client.edict = ent
  client.message points at the client's own buffer, ALLOWING OVERFLOW
  client.privileged = false
  IF restoring a saved game
    put the saved carry values back
  ELSE
    run the game's SetNewParms ;  copy the sixteen globals into the slot
  sv_send_serverinfo(client)
```

**Invariants** — the reliable buffer **permits overflow**, which means an overflow clears it and
sets a flag rather than aborting ([`common.c`](common.c.md#sz_getspace)). The comment says "we
can catch it", and [`SV_SendClientMessages`](#sv_sendclientmessages) is where it is caught, by
dropping the client. A rebuild must keep the pairing: tolerating the overflow without the later
check silently loses reliable messages.

The privilege flag is set to false unconditionally in this build. A build switch replaces it with
an address test against a fixed list — the source's own comment elsewhere calls that switch very
dangerous. See [`sv_user.c`](sv_user.c.md#the-authorization-list) for what the flag grants.

Called **once per player per game**, not once per level: a level change reuses the slot and only
resends the server information.

### `SV_CheckForNewClients`

**Contract** — accepts every pending connection into the first free slot. No free slot is a fatal
error.

**Notes** — the error is fatal because the network layer accepted a connection the server cannot
house, which means the two disagree about the limit. A rebuild should refuse the connection
instead.

## Visibility

### `SV_AddToFatPVS`

**Contract** — walks the rendering tree from a node, accumulating the visibility sets of every
leaf whose plane distance from the given point is within eight units on either side. Solid leaves
contribute nothing.

```text
FUNCTION sv_add_to_fat_pvs(org, node)
  LOOP
    IF node IS a leaf
      IF the leaf is not solid
        union its visibility set INTO fatpvs
      RETURN
    d = signed distance FROM org TO node's plane
    IF d >  8  node = node.children[0]          # clearly in front
    ELSE IF d < -8  node = node.children[1]     # clearly behind
    ELSE
      recurse INTO children[0] ;  node = children[1]   # within 8: BOTH sides
```

**Invariants** — the eight-unit slack is the whole point, and the source explains it: the client's
view bobs as the player walks, and an entity near a leaf boundary would appear and disappear as
the bob crossed it — "especially when the bob crosses a waterline", where the two leaves have very
different visibility. Widening the query by the bob's amplitude fixes it at the cost of sending a
few more entities.

The traversal is a loop with one recursion, descending the second child iteratively — an
optimization, not a decision.

### `SV_FatPVS`

**Contract** — computes the fattened visibility set for a point and returns it.

```text
FUNCTION sv_fat_pvs(org) -> bytes
  fatbytes = (the leaf count + 31) SHIFTED RIGHT 3
  zero fatbytes bytes of fatpvs
  sv_add_to_fat_pvs(org, the world's root node)
  RETURN fatpvs
```

**Notes** — the byte count rounds the *bit* count up to a multiple of 32 bits and then divides by
8, so it allocates up to three bytes more than needed. Deliberate padding for the word-wise
clearing, and harmless. A rebuild should round to 8.

### `SV_WriteEntitiesToClient`

**Contract** — appends an update for every entity that might be visible to a client, plus the
client's own entity unconditionally. Skips entities with no visible model, and entities whose leaf
list does not intersect the fattened visibility set. Stops and prints if the message fills up. Each
update is a flag word plus the fields that differ from that entity's spawn baseline.

```text
FUNCTION sv_write_entities_to_client(clent, msg)
  pvs = sv_fat_pvs(clent.origin + clent.view_ofs)     # from the EYES

  FOR EACH entity ent, number e, FROM 1 upward
    IF ent IS NOT clent                               # the client is always sent
      IF ent has no model index OR its model name is empty  CONTINUE
      IF none OF ent's touched leaves is set IN pvs          CONTINUE
    IF the message has under 16 bytes free
      print "packet overflow" ;  RETURN               # the rest are LOST

    bits = 0
    FOR EACH axis i
      IF |ent.origin[i] - ent.baseline.origin[i]| > 0.1  set the origin-i bit
    FOR EACH axis i
      IF ent.angles[i] != ent.baseline.angles[i]         set the angle-i bit
    IF ent.movetype IS step      set the no-interpolate bit
    IF colormap, skin, frame, effects, modelindex differ FROM THE BASELINE
      set each corresponding bit
    IF e >= 256    set the long-entity bit
    IF bits >= 256 set the more-bits bit

    write (bits BITOR the signal bit) as a byte
    IF more-bits     write the high byte of bits
    write e as a short IF long-entity ELSE as a byte
    write, IN BIT ORDER: modelindex, frame, colormap, skin, effects,
           then INTERLEAVED origin[0], angle[0], origin[1], angle[1],
           origin[2], angle[2]
```

**Invariants** — five things here define the protocol's character.

**The comparison is against the spawn baseline, not against what this client last received.** So
the server keeps no per-client history and every frame is self-describing *relative to the
baseline* — which the client received once, reliably, at connect. The consequence: a client that
receives a frame has correct state for every entity in it, and an entity omitted because it left
the visibility set simply stays where it was. That is why entities in this game freeze rather than
disappear when you turn away.

**A lost packet loses that frame entirely and nothing recovers it**, because the next frame
describes the same baseline delta and not the missed change. For an entity that moved once and
stopped, the client is permanently wrong until the entity moves again. Over a local network,
where loss is near zero, this is fine. Over the internet it is not, and it is exactly what
[`QW`](../QW/server/sv_ents.c.md) replaces with per-client frame history.

**The origin threshold is a tenth of a unit and the angle threshold is exact equality.** So an
entity whose angle changed by a thousandth of a degree costs a byte every frame. The asymmetry is
deliberate for positions, where floating-point noise is expected, and an oversight for angles.

**Stepped entities are always marked non-interpolatable.** A monster moves in discrete steps
([`sv_move.c`](sv_move.c.md)), so interpolating between two of its positions would slide it
smoothly where it should snap. Setting the bit unconditionally for that movement type is the
cheapest correct answer.

**The field order on the wire is the bit order**, with the coordinates and angles *interleaved*
per axis. A reader must consume them in exactly that order.

**The overflow behaviour is silent truncation of the remaining entities.** Entities later in the
array are dropped for this frame, so **entity index order is a visibility priority** — low-numbered
entities are favoured under load. Players are entities 1 through 16, so players are never the
ones dropped. That is accidental and fortunate.

### `SV_CleanupEnts`

**Contract** — clears the muzzle-flash effect bit on every entity. Called once per frame after
messages are sent.

**Invariants** — the muzzle flash is a **one-frame** effect and this is how it is made one: the
game sets the bit, the frame's messages carry it, and this clears it. A rebuild that omits this
leaves every fired weapon permanently glowing.

## Client state

### `SV_WriteClientdataToMessage`

**Contract** — appends the client's own state: a damage message if they were hurt, an absolute
view angle if game logic demanded one, then a flagged record of view height, ideal pitch, view
kick, velocity, items, ground and water state, weapon animation frame, armour, weapon model,
health and five ammo counts.

```text
FUNCTION sv_write_clientdata_to_message(ent, msg)
  IF ent took damage this frame
    write the damage tag, the armour absorbed, the health lost, and the
          bounding-box CENTRE of whatever inflicted it
    clear both damage fields
  sv_set_ideal_pitch()                              # compute the stair tilt

  IF ent.fixangle
    write the set-angle tag and all three angles ;  clear fixangle
    # "a fixangle might get lost in a dropped packet. Oh well."

  bits = 0
  IF ent.view_ofs[2] != 22          set the view-height bit
  IF ent.idealpitch  != 0           set the ideal-pitch bit

  # Two 32-bit item sets do not fit in one. If the game declares a second set,
  # pack it into the high bits; otherwise pack the episode flags there.
  items = ent.items
  val = the value of ent's "items2" field, IF the game declared one
  IF val EXISTS  items = items BITOR (val SHIFTED LEFT 23)
  ELSE           items = items BITOR (the global serverflags SHIFTED LEFT 28)

  set the items bit ALWAYS
  IF ent is on the ground           set the on-ground bit    # no payload
  IF ent.waterlevel >= 2            set the in-water bit     # no payload
  FOR EACH axis i
    IF ent.punchangle[i] != 0       set the punch-i bit
    IF ent.velocity[i]   != 0       set the velocity-i bit
  IF ent.weaponframe != 0           set the weapon-frame bit
  IF ent.armorvalue  != 0           set the armour bit
  set the weapon bit ALWAYS

  write the clientdata tag and bits as a short
  write, IN BIT ORDER: view height (a signed byte), ideal pitch (a signed byte),
        then per axis the punch angle (a signed byte) and the velocity DIVIDED
        BY 16 (a signed byte), then items (a long), the weapon frame, the
        armour value, the weapon model's index
  write health (a short), current ammo, and the four ammo counts (bytes)
  IF this is the original game
    write ent.weapon as a byte
  ELSE
    write the INDEX of ent.weapon's lowest set bit as a byte
```

**Invariants** — several load-bearing oddities.

**The item set is 32 bits and two things want the top of it.** The status bar needs to know which
episode sigils have been collected, and a mission pack needs a second item set. The engine packs
whichever is present into the high bits — the second set at bit 23, the episode flags at bit 28 —
so the client's status bar reads a *composite* field whose meaning depends on which game is
loaded. A rebuild must send both halves and must agree with [`sbar.c`](sbar.c.md) about the
packing.

**Velocity is divided by 16 into a signed byte**, so the client knows its own velocity only to
±8 units per second and only up to ±2048. It is used for the view bob and the fall sound, which
is all that resolution supports.

**Two flags carry no payload.** The ground and water bits *are* their values.

**The items and weapon bits are always set**, so those two fields are always present despite
being flagged. The source has a comment marking the items case as always sent and a commented-out
condition on the weapon one. A rebuild should send them unconditionally and not flag them, but
must still set the bits or a matching client will misparse.

**The weapon field is transmitted differently per game variant.** The original sends a model
index; the mission packs send the *bit position* of the weapon's lowest set bit, because their
weapon field is a bit set rather than an index. Two wire meanings for one field, selected by a
startup flag ([`common.c`](common.c.md#com_initargv)).

**An absolute-angle command is sent once and not retransmitted**, and the source shrugs about it.
So a teleport that lands in a dropped packet leaves the player facing the wrong way. A rebuild
should send it reliably.

The stair-tilt computation ([`sv_user.c`](sv_user.c.md#sv_setidealpitch)) is invoked from the
*message writer*, which is a layering oddity: it performs several sweeps as a side effect of
composing a packet. A rebuild should compute it during movement.

### `SV_SendClientDatagram`

**Contract** — composes and sends one client's unreliable frame: the server time, their own state,
the visible entities, then as much of the frame's broadcast as still fits. Drops the client if the
send fails.

```text
FUNCTION sv_send_client_datagram(client) -> bool
  msg = a fresh 1024-byte buffer
  write the time tag and sv.time as a float          # ALWAYS first
  sv_write_clientdata_to_message(client.edict, msg)
  sv_write_entities_to_client(client.edict, msg)
  IF the broadcast datagram still fits  append it whole
  IF sending unreliably failed  sv_drop_client(crashed = true) ;  RETURN false
  RETURN true
```

**Invariants** — the **time leads every frame**, which is what lets the client interpolate
([`protocol.h`](protocol.h.md)).

The broadcast is appended **whole or not at all**. So a frame with many visible entities can drop
every particle, sound and effect for that client while another client receives them. That is
observable: under load, sounds go missing for the players who can see the most.

The per-client buffer is a **stack local**, so nothing accumulates between frames — which is the
definition of unreliable here.

### `SV_UpdateToReliableMessages`

**Contract** — appends to every client's reliable stream: a score update for each player whose
score changed, and then the frame's reliable broadcast. Clears the broadcast.

**Invariants** — score changes are detected by comparing against a remembered value per client,
which is the *only* per-client delta state in this protocol. Everything else compares against a
baseline.

The reliable broadcast is copied into each client's own stream rather than sent separately,
because the reliable layer is per-connection ([`net.h`](net.h.md)).

### `SV_SendNop`

**Contract** — sends a single do-nothing message unreliably, without touching the client's
accumulated reliable buffer. Records the send time. Drops the client on failure.

**Invariants** — the point is to keep a connection alive during a long level load without
flushing a reliable stream that is still being built. Its own tiny buffer is what makes that
possible.

### `SV_SendClientMessages`

**Contract** — the per-frame send. Updates the reliable streams, then for each active client:
sends the unreliable frame if they are in the world, or a keepalive if they have been quiet for
five seconds during the handshake; drops any client whose reliable buffer overflowed; and flushes
the reliable buffer if there is anything in it and the channel will accept it. Finally clears the
one-frame effects.

```text
FUNCTION sv_send_client_messages()
  sv_update_to_reliable_messages()
  FOR EACH active client
    IF spawned
      IF NOT sv_send_client_datagram(client)  CONTINUE     # it dropped them
    ELSE IF NOT client.sendsignon
      IF more than 5 seconds since the last message  sv_send_nop(client)
      CONTINUE                                 # send nothing else mid-handshake

    IF client.message.overflowed
      sv_drop_client(crashed = true) ;  clear the flag ;  CONTINUE

    IF client.message has content OR client.dropasap
      IF the reliable channel will not accept a message now  CONTINUE
      IF client.dropasap
        sv_drop_client(crashed = false)         # they changed level
      ELSE
        IF sending reliably failed  sv_drop_client(crashed = true)
        clear client.message
        client.last_message = now ;  client.sendsignon = false
  sv_cleanup_ents()
```

**Invariants** — a client mid-handshake receives **only** signon messages and keepalives, which is
what keeps their reliable buffer from being flushed with a partial handshake in it.

A **reliable overflow drops the client**, which is the promised catch for the overflow-tolerant
buffer. The source's comment says it should only happen on a badly backed-up connection that then
changes level. That is the correct response: the client has provably missed reliable data and
cannot be repaired.

The reliable buffer is flushed **only when the channel says it will accept a message**
([`net.h`](net.h.md)), because the reliable layer holds one outstanding message per direction.
So a client with a slow link accumulates reliable data across frames, which is why the buffer can
overflow at all.

`dropasap` is checked **before** sending, so a client told to change level is dropped rather than
sent more data.

## Level spawning

### `SV_ModelIndex`

**Contract** — takes a model name; returns its precache index, or 0 for an empty name. An
unprecached name is a fatal error.

### `SV_CreateBaseline`

**Contract** — records every live entity's current state as its baseline and appends that baseline
to the signon stream. Skips free entities, and skips non-player entities with no model. Players
get a fixed model and a colour map equal to their entity number.

```text
FUNCTION sv_create_baseline()
  FOR EACH entity number entnum FROM 0 upward
    svent = entity entnum
    IF svent IS free  CONTINUE
    IF entnum is above the player range AND svent has no model index  CONTINUE

    svent.baseline.origin = svent.origin
    svent.baseline.angles = svent.angles
    svent.baseline.frame  = svent.frame
    svent.baseline.skin   = svent.skin
    IF entnum IS a player slot
      svent.baseline.colormap   = entnum                 # their player colours
      svent.baseline.modelindex = the index of the player model
    ELSE
      svent.baseline.colormap   = 0
      svent.baseline.modelindex = the index of svent.model

    append to the signon: the baseline tag, entnum as a short, then the model
           index, frame, colormap and skin as bytes, then per axis the
           coordinate and the angle INTERLEAVED
```

**Invariants** — a player's colour map is their entity number, which is how the client knows which
palette translation to apply to which player model
([`r_alias.c`](r_alias.c.md)). A non-player's is zero, meaning no translation.

The player model is looked up **by name**, so it must be precached by the game logic — and it is,
because the game precaches it in its world spawn function.

The baseline is recorded **after** two physics frames have run (below), so it is the world's
settled state rather than its authored state. That matters: items have fallen to their floors and
doors have found their closed positions.

### `SV_SendReconnect`

**Contract** — tells every client, reliably and blocking, to run the reconnect command; then runs
it locally unless this is a dedicated server.

**Invariants** — the level change is delivered as a **stuffed console command**
([`protocol.h`](protocol.h.md)), not as a protocol message. So the mechanism by which every
client re-enters the handshake is "the server types `reconnect` on their console". Sent through
the blocking broadcast ([`net.h`](net.h.md#net_sendtoall)) with a five-second budget, because a
client that misses it is stranded.

### `SV_SaveSpawnparms`

**Contract** — records the episode flags and, for each active player, asks the game to fill the
sixteen carry values and stores them in the player's slot.

**Invariants** — this is the entire mechanism by which a player keeps their inventory across a
level. The engine does not know what the values mean; the game writes them on the way out
([`progdefs.q1`](progdefs.q1.md)) and reads them on the way in.

### `SV_SpawnServer`

**Contract** — starts a level. Tells existing clients to reconnect, reconciles the game-mode
variables, discards all memory, loads the game logic, allocates the entity array, sets up the
three broadcast buffers, reserves the player entity slots, loads the map and its sub-models,
builds the collision index, populates the precache lists, initializes the world entity, spawns
every map entity, runs two physics frames, records the baselines, and sends the server information
to everyone connected.

```text
FUNCTION sv_spawn_server(server)
  IF the host name is empty  set it to "UNNAMED"
  reset the centre-print timer
  svs.changelevel_issued = false            # another change is now permitted
  IF a server is already active  sv_send_reconnect()

  IF cooperative  force deathmatch off
  current_skill = round(the skill variable), CLAMPED to 0..3
  write the clamped value back to the variable

  host_clear_memory()                       # discards the previous level
  zero the whole per-level record
  sv.name = server

  pr_load_progs()                           # FIRST: it decides the entity size
  sv.max_edicts = 600
  sv.edicts = hunk-allocate 600 * the entity stride
  point the three broadcast buffers at their inline storage

  sv.num_edicts = maxclients + 1            # reserve the low slots for players
  FOR EACH player slot i  bind it to entity (i+1)
  sv.state = loading ;  sv.paused = false
  sv.time = 1.0                             # NOT zero; see the note

  sv.modelname = "maps/<server>.bsp"
  sv.worldmodel = load it
  IF it failed  print "Couldn't spawn server" ;  sv.active = false ;  RETURN
  sv.models[1] = the world model

  sv_clear_world()                          # build the collision index

  sv.sound_precache[0] = the empty string
  sv.model_precache[0] = the empty string
  sv.model_precache[1] = sv.modelname
  FOR EACH sub-model i FROM 1
    sv.model_precache[1+i] = the name "*i"
    sv.models[i+1] = load that sub-model

  # The world entity
  ent = entity 0
  zero its interpreter fields ;  mark it in use
  ent.model = the world model's name ;  ent.modelindex = 1
  ent.solid = solid_bsp ;  ent.movetype = push

  IF cooperative  the global coop = 1  ELSE  the global deathmatch = its value
  the global mapname = sv.name
  the global serverflags = svs.serverflags       # carried episode progress

  ed_load_from_file(the map's entity text)       # spawns everything

  sv.active = true
  sv.state = active                 # further precache calls are now errors

  host_frametime = 0.1
  sv_physics() ;  sv_physics()       # two frames, to let everything settle

  sv_create_baseline()
  FOR EACH active client  sv_send_serverinfo(client)
```

**Invariants** — the order here is the level-load contract and six steps cannot move.

**The game logic loads before the entity array is allocated**, because it decides how large an
entity is ([`progs.h`](progs.h.md)).

**The clock starts at 1.0, not 0.** Game logic tests `nextthink` against the current time and
treats zero as "never", so a level starting at time zero would make every entity's initial
schedule ambiguous. Starting at 1 gives a full second of headroom.

**The player slots are reserved by setting the entity count to the player limit plus one**, before
any map entity is spawned. That is what makes player *n* always entity *n+1*
([`pr_edict.c`](pr_edict.c.md#ed_alloc) then allocates above the reserved range).

**The precache lists begin with an empty string at index 0**, so index 0 means "none" for both
models and sounds, and index 1 is the map itself.

**Sub-models are precached and loaded before the map's entities spawn**, because a door's spawn
function immediately sets its model to `*3`.

**Two physics frames run at a fixed tenth of a second before the baselines are recorded.** That is
what lets items fall to the floor and doors settle, so the baselines describe a stable world and
the first frame's deltas are nearly empty. A rebuild that skips this sends a large first frame and
shows items floating for a moment.

The state moves from loading to active *before* the physics frames, so those two frames run under
the same rules as normal play — and any precache call from a think function during them is an
error ([`pr_cmds.c`](pr_cmds.c.md#precaching)).

The level name is assigned twice, harmlessly.

**Notes** — the entity array is sized to the hard maximum rather than to the level's needs, which
with a 90-field game is about 250 kilobytes. Fixed cost per level.
