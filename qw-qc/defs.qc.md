# qw-qc/defs.qc

> The interface between the engine and the game: the globals the engine sets, the callbacks it calls, every field an entity has, and every operation the game may ask of the engine.

**Needs** — nothing; it is the first file compiled
**Used by** — every other file in this chapter, and — as a generated header — the engine ([`progdefs.h`](../QW/server/progdefs.h.md))
**Tier floor** — none

## Purpose

The contract. Read this before anything else in the chapter, and read it beside
[`progdefs.h`](../QW/server/progdefs.h.md), which the game-logic compiler generates from the first part of it and the engine then
compiles in. The two are the same declarations from opposite sides.

Its structure is four sections in a fixed order, and the order is load-bearing: **the compiler emits everything before a marker as the
engine's shared globals, and everything before a second marker as the engine's shared entity fields.** Anything after a marker is the
game's own and the engine does not know it.

## State

The shared globals and entity fields are the state, and they are enumerated in the sections below.

## The shared globals

```text
self, other, world              # entity references the engine sets before a call
time, frametime                 # the clock the game sees
newmis                          # set by the game when it creates a projectile,
                                #   read by the engine to simulate it at once
force_retouch                   # a countdown: make everything touch triggers
mapname, serverflags
total_secrets, total_monsters, found_secrets, killed_monsters
parm1 .. parm16                 # per-player state carried across levels
v_forward, v_up, v_right        # set by the basis-construction operation
trace_allsolid, trace_startsolid, trace_fraction, trace_endpos,
  trace_plane_normal, trace_plane_dist, trace_ent, trace_inopen, trace_inwater
msg_entity                      # the recipient of a directed message
END MARKER
```

**Invariants** —

- **The trace's results are nine globals, not a return value**, because the language has no records. So a trace's answer is valid only
  until the next trace, and interleaving two traces is a bug the language cannot prevent. Every caller in this chapter reads the results
  immediately.
- **The basis vectors are globals set by an operation**, for the same reason.
- **The projectile signal is the only global the engine reads *after* calling the game** rather than before
  ([`sv_phys.c`](../QW/server/sv_phys.c.md)), which is why a rocket fired this frame moves this frame.
- **The retouch flag is a countdown, not a boolean**, and the comment explains why: a newly created trigger must catch things that are
  not moving, and one frame is not enough because the order in which entities run is not controlled. Setting it to two guarantees
  coverage. That is a real subtlety about a trigger system built on movement.
- **Sixteen numbers are the whole of per-player persistence across levels**
  ([`sv_init.c`](../QW/server/sv_init.c.md)), and the game encodes health, armour, weapons and ammunition into them
  ([`client.qc`](client.qc.md)).

## The callbacks

```text
main               # unused, for testing
StartFrame         # once per server frame, before anything moves
PlayerPreThink     # per player, before their movement runs
PlayerPostThink    # per player, after it
ClientKill         # the player asked to die
ClientConnect      # a player joined
PutClientInServer  # spawn them; called after their parms are set
ClientDisconnect   # a player left
SetNewParms        # initialize a new player's persistent parms
SetChangeParms     # save a player's parms for a level change
```

**Invariants** — **this list is the whole of the game's lifecycle**, and it is worth reading as the specification of *when a rule may
run*. Note what is absent: there is no per-entity frame callback — an entity runs only when its scheduled thinking time arrives — and
no callback for a level ending. QuakeWorld adds a spectator variant of the think callbacks
([`spectate.qc`](spectate.qc.md)).

## The shared entity fields

```text
*** = maintained by the engine, the game must not set

*** modelindex, absmin, absmax, origin, oldorigin, lastruntime
    ltime                       # this entity's own clock, for pushers
    movetype, solid             # how the engine simulates and collides it
    velocity, angles, avelocity
    classname                   # which spawn function the map's entity chose
    model, frame, skin, effects
    mins, maxs, size            # the collision box
    touch, use, think, blocked  # the four handlers the engine calls
    nextthink                   # when to call think
    groundentity
    health, frags, weapon, weaponmodel, weaponframe, currentammo,
      ammo_shells/nails/rockets/cells, items, armortype, armorvalue,
      max_health, takedamage, deadflag
    chain                       # the engine's field-search result list
    view_ofs                    # eye position relative to origin
    button0, button1, button2, impulse    # what the player is pressing
    fixangle, v_angle           # forcing and reading the view angle
    netname, team, colormap
    enemy, aiment, goalentity   # references the game uses, engine-visible
    flags
    teleport_time               # doubles as the water-jump timer (pmove.c)
    waterlevel, watertype
    ideal_yaw, yaw_speed        # for the engine's monster turning
    spawnflags                  # the map's per-entity option bits
    ... and the remaining shared fields
END MARKER
```

**Invariants** —

- **The engine-maintained fields are marked and must not be written by the game.** Writing the origin directly instead of through the
  engine's placement operation leaves the entity unlinked from the collision world, which is the single most common mistake a
  modification makes. The convention is a comment; a rebuild should make it enforceable.
- **Four handler fields — touch, use, think, blocked — are the entire callback surface of an entity.** Everything the game does is
  scheduled through them, and `nextthink` is the only timer.
- **The statistics fields are shared because the engine sends them to the client**
  ([`sv_send.c`](../QW/server/sv_send.c.md)), not because the engine uses them. So the game's notion of health and ammunition is in the
  engine's record purely for transmission, which is the layering compromise
  [`bothdefs.h`](../QW/client/bothdefs.h.md) also records.
- **One field is reused for two purposes**: the teleport timer is also the movement model's water-jump timer
  ([`pmove.c`](../QW/client/pmove.c.md)). That aliasing is deliberate and it is the kind of thing that must be written down, because
  nothing in either file says so.
- **The field order is the contract** ([`progdefs.h`](../QW/server/progdefs.h.md)) and is verified by a checksum at load, which is why
  game logic compiled for the original engine is refused.

## The engine operations

**Contract** — the numbered list of everything the game may ask the engine to do: vector arithmetic and normalization, the basis
construction, random numbers, tracing, placing an entity, spawning and removing, sound, printing to one client or all, precaching,
the field search, the string operations, the entity-to-text conversions, the movement helpers, the message composition and routing, the
dictionary read, the kill log, and the trigonometric functions.

**Invariants** —

- **Each operation is bound to a number and a number's meaning can never change.** The numbering is compiled into the game logic, so
  the engine's table ([`pr_cmds.c`](../QW/server/pr_cmds.c.md)) is append-only forever. That is the single hardest constraint in the
  whole interface.
- **Trigonometry, square root and string-to-number are engine operations**, because the language has arithmetic and comparison and
  nothing else.
- **Message composition is a byte-at-a-time interface followed by a routing call**
  ([`pr_cmds.c`](../QW/server/pr_cmds.c.md)), and the pair must be used together.

## The game's own declarations

After the second marker: the item flags, the damage and death constants, the animation frame names, the sound names, the class
constants, the temporary-entity kinds mirrored from the protocol, and the forward declarations every later file needs.

**Invariants** — **the temporary-entity kinds are duplicated from the protocol header**
([`protocol.h`](../QW/client/protocol.h.md)) and the source says so. Two hand-synchronized copies of one numbering; a rebuild
generates one from the other.

**Notes** — the most useful thing about this file for a rebuilder is the *shape* of the contract: a fixed set of globals the engine
writes before each call, four handler fields per entity, one timer, ten lifecycle callbacks, and a numbered append-only operation table.
That is a complete and surprisingly small interface for a scripting boundary, and the parts that are awkward — traces returning through
globals, the operation numbering being permanent, statistics living in the engine's record — are each awkward for a reason a rebuild can
address directly.
