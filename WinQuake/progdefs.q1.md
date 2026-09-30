# WinQuake/progdefs.q1

> Generated: the exact slot-by-slot layout of the globals and entity fields that the engine and the game logic must agree on, plus the checksum that enforces the agreement.

**Needs** — [`mathlib.h`](mathlib.h.md) (the vector type) · [`pr_comp.h`](pr_comp.h.md) (the string and function handle types)
**Used by** — [`progdefs.h`](progdefs.h.md) selects it; [`progs.h`](progs.h.md) embeds both records; the server reads and writes nearly every field in it
**Tier floor** — T1 as written, because the engine overlays a native record on interpreter memory. A rebuild that accesses these fields by computed offset instead reaches T3.

## Purpose

This file is the interface between the engine and the game, and it is *generated* — by
the game-logic compiler, from the game's own declarations, and then checked in. That
direction matters: the game defines the layout and the engine conforms, not the other
way round.

What it buys is that the server can read `entity.origin` as a field of a native record
rather than looking up a field offset. What it costs is that **changing the game's field
declarations changes the engine's layout**, which is why the checksum at the bottom
exists and why a mismatch is fatal.

## State

Two record layouts and one constant. No behaviour.

## The globals

```text
RECORD GlobalVars              # overlaid on the interpreter's global slot pool
  pad : int[28]                # the call frame: null, return, eight arguments
                               # (pr_comp.h's reserved_ofs)

  # --- what the engine writes and the game reads ---
  self, other, world : entity  # the entity a callback is about; its partner;
                               # entity 0
  time               : real    # the server's clock, in seconds
  frametime          : real    # the current server tick's duration
  force_retouch      : real    # a countdown: while positive, re-run every
                               # trigger overlap next frame
  mapname            : string
  deathmatch, coop, teamplay : real
  serverflags        : real    # which episode sigils have been collected;
                               # survives level changes
  total_secrets, total_monsters : real
  found_secrets, killed_monsters : real
  parm1 .. parm16    : real    # the sixteen slots that carry a player's
                               # inventory across a level change
  v_forward, v_up, v_right : vec3   # filled in by the host function that
                                    # converts an angle triple to a basis
  trace_allsolid, trace_startsolid, trace_fraction : real
  trace_endpos, trace_plane_normal : vec3
  trace_plane_dist   : real
  trace_ent          : entity
  trace_inopen, trace_inwater : real     # the result of the last line trace
  msg_entity         : entity  # the recipient of the next message write

  # --- entry points the engine CALLS, by name resolved at load ---
  main               : func    # unused in the shipped game
  StartFrame         : func    # once per server tick, before any thinks
  PlayerPreThink     : func    # per player, before physics
  PlayerPostThink    : func    # per player, after physics
  ClientKill         : func    # the player asked to die
  ClientConnect      : func
  PutClientInServer  : func    # spawn or respawn
  ClientDisconnect   : func
  SetNewParms        : func    # fill the sixteen carry slots for a fresh start
  SetChangeParms     : func    # fill them for a level change
```

**Invariants** — the 28-slot pad at the front is not alignment. It is the interpreter's
fixed call frame ([`pr_comp.h`](pr_comp.h.md)), which the compiler places first, so the
engine's overlay must skip it to reach the first real global.

The trace result is **returned through globals**, not as a value: the host function that
traces a line writes eight globals and the game reads them. That is forced by the
language having no records and no multiple return, and it means a trace cannot be nested
— the game must consume the result before tracing again. Every piece of game logic is
written around that.

The sixteen carry slots are the entire mechanism by which a player keeps their weapons
across a level. The game fills them on the way out and reads them on the way in; the
engine only stores them, in the save file and across the level load. Sixteen is arbitrary
and fixed on both sides.

`force_retouch` is a countdown rather than a flag so that the game can request "re-run
every trigger overlap for the next two frames", which it needs after teleporting or
spawning something inside a trigger.

## The entity fields

```text
RECORD EntVars                 # the first `n` slots of every entity
  modelindex        : real     # index into the level's model list
  absmin, absmax    : vec3     # world-space bounding box, engine-maintained
  ltime             : real     # a moving brush's own clock
  movetype          : real     # which physics rule applies
  solid             : real     # how this entity collides
  origin            : vec3
  oldorigin         : vec3
  velocity          : vec3
  angles            : vec3
  avelocity         : vec3     # angular velocity
  punchangle        : vec3     # a view kick, for players
  classname         : string   # the map's spawn key; also the game's type tag
  model             : string
  frame             : real     # animation frame
  skin              : real
  effects           : real     # a bit set the engine turns into visual effects
  mins, maxs        : vec3     # the bounding box, relative to the origin
  size              : vec3     # maxs minus mins; engine-maintained
  touch, use, think, blocked : func    # the four callbacks
  nextthink         : real     # the time at which `think` should run
  groundentity      : entity   # what this is standing on, or nothing
  health, frags, weapon : real
  weaponmodel       : string
  weaponframe, currentammo : real
  ammo_shells, ammo_nails, ammo_rockets, ammo_cells : real
  items             : real     # the item bit set
  takedamage        : real     # 0 no, 1 armour applies, 2 always
  chain             : entity   # a scratch field the game uses to build
                               # linked lists from a spatial query
  deadflag          : real
  view_ofs          : vec3     # eye position relative to the origin
  button0, button1, button2 : real    # attack, use, jump — held state
  impulse           : real     # a one-shot command number from the player
  fixangle          : real     # the engine should send an absolute view angle
  v_angle           : vec3     # a player's actual view angles
  idealpitch        : real     # the pitch a monster or the auto-aim wants
  netname           : string
  enemy             : entity
  flags             : real     # engine and game flags: on ground, in water,
                               # is a monster, is a client, is godmode-immune
  colormap, team, max_health : real
  teleport_time     : real
  armortype, armorvalue : real
  waterlevel, watertype : real # engine-maintained: how deep, in what
  ideal_yaw, yaw_speed : real  # monster turning
  aiment            : entity   # what this entity is attached to
  goalentity        : entity
  spawnflags        : real     # from the map
  target, targetname : string  # the map's entity-to-entity wiring
  dmg_take, dmg_save : real    # damage taken this frame, for the view flash
  dmg_inflictor     : entity
  owner             : entity   # who fired this projectile; excluded from
                               # its collisions
  movedir           : vec3
  message           : string
  sounds            : real
  noise, noise1, noise2, noise3 : string
```

**Invariants** — several of these are *engine-maintained*: the world-space bounding box,
the size, the water level and type, the ground entity, and the flags for on-ground and
in-water. The game reads them and must not write them, and the engine recomputes them
whenever an entity is moved or resized
([`world.c`](world.c.md#sv_linkedict), [`sv_phys.c`](sv_phys.c.md)).

The `chain` field is a scratch slot: the host function that finds every entity within a
radius threads its results through it, and the game walks the chain. So **only one such
query can be live at a time**, and nesting two loses the outer one. The same pattern
appears in the find-by-field host function. A rebuild with real collections should still
expose the chain, because published game logic uses it.

`aiment` attaches one entity's motion to another's and is how a platform carries a
player and how the chase camera follows. The attachment is resolved by the physics code,
not here.

Entity 0 is the world, and assigning to any field of it while a level is running is a
run-time error the interpreter raises by name
([`pr_exec.c`](pr_exec.c.md)).

**Notes** — **this record is a prefix, not the whole entity.** The game may declare any
number of further fields; they occupy slots after these and are reachable only by field
offset, never as native fields. The count comes from the program header
([`pr_comp.h`](pr_comp.h.md)) and sets the entity stride. So the engine sees the first
~90 slots as a record and the rest as an opaque tail — which is exactly what makes game
modifications possible without recompiling the engine.

## The checksum

```text
CONSTANT progheader_crc = 5927
```

**Invariants** — a CCITT-16 checksum over the *text of the declarations above*, computed
by the compiler and by the engine independently ([`pr_edict.c`](pr_edict.c.md#pr_loadprogs))
and compared. A mismatch refuses to load the game logic at all.

It is also sent to clients in the server-information message
([`protocol.h`](protocol.h.md)), because a client reads some of these fields for its own
prediction and must agree about their offsets.

**Notes** — the check is a layout check, not a version check and not a tamper check. A
game modification that adds fields at the end keeps the same checksum and loads fine;
one that reorders or inserts does not. That is the correct boundary, and it is why the
original's modding community could ship new game logic without shipping an engine.

## Why the file is generated

A rebuild does not need to generate it. What it needs is the property the generation
guarantees: **the engine's view of an entity's first *n* fields must be byte-identical
to the compiler's**. A rebuild that reads fields through the definition tables by name
at load time — resolving `origin` to a slot offset once — gets the same result with no
generated file and no checksum, at the cost of one indirection per access. That is the
change most worth making.
