# WinQuake/server.h

> The server's two records — one per process, one per level — the per-player record, and the enumerations that name every way an entity can move and collide.

**Needs** — [`common.h`](common.h.md) (the byte buffer) · [`progs.h`](progs.h.md) · [`net.h`](net.h.md) · [`protocol.h`](protocol.h.md) (the movement command) · [`cvar.h`](cvar.h.md)
**Used by** — [`sv_main.c`](sv_main.c.md) · [`sv_phys.c`](sv_phys.c.md) · [`sv_move.c`](sv_move.c.md) · [`sv_user.c`](sv_user.c.md) · [`world.c`](world.c.md) · [`pr_cmds.c`](pr_cmds.c.md) · [`pr_edict.c`](pr_edict.c.md) · [`host.c`](host.c.md) · [`host_cmd.c`](host_cmd.c.md) · [`quakedef.h`](quakedef.h.md) includes it for everyone
**Tier floor** — none

## Purpose

Two things live here. The state layout — which is a clean statement of what survives a level
change and what does not — and the enumerations, which are the vocabulary the game logic and
the physics share. The enumerations are the more important half: the movement types are a
*closed set of physics behaviours*, each implemented by a branch in
[`sv_phys.c`](sv_phys.c.md), and the solidity types are a closed set of collision
behaviours. Between them they are the whole interface between the game's intentions and the
engine's simulation.

## State

```text
RECORD ServerStatic              # survives a level change
  maxclients      : int          # how many player slots exist
  maxclientslimit : int          # how many were allocated; never shrinks
  clients         : list<Client> # the player slots
  serverflags     : int          # episode completion; carried between levels
  changelevel_issued : bool      # cleared when a level starts

RECORD Server                    # rebuilt for every level
  active   : bool                # false when this process is only a client
  paused   : bool
  loadgame : bool                # a restored game: handle connections
                                 # differently
  time     : real                # the simulation clock, from zero at level start

  lastcheck     : int            # the awareness cursor, see
  lastchecktime : real           # pr_cmds.c's checkclient

  name          : text[64]       # the map's short name
  modelname     : text[64]       # its full asset path; model index 0
  worldmodel    : Model
  model_precache : text[256]     # the level's model list, terminated by an
                                 # empty entry
  models         : Model[256]    # the loaded models, by the same index
  sound_precache : text[256]     # the level's sound list, likewise
  lightstyles    : text[64]      # the animation string per light style
  num_edicts, max_edicts : int
  edicts    : Edict[]            # NOT indexable directly: the stride is a
                                 # runtime value (progs.h)
  state     : enum { loading, active }   # some actions are legal only
                                         # while loading

  datagram          : SizeBuf    # this frame's UNRELIABLE broadcast
  reliable_datagram : SizeBuf    # this frame's RELIABLE broadcast
  signon            : SizeBuf    # the connection preamble; 8192 bytes

RECORD Client                    # one player slot
  active     : bool              # the slot is occupied
  spawned    : bool              # the player is in the world; send them
                                 # datagrams
  dropasap   : bool              # told to change level; drop after the message
                                 # goes out
  privileged : bool              # may run any console command
  sendsignon : bool              # meaningful only before spawning
  last_message : real            # when a reliable message last went out
  netconnection : QSocket
  cmd      : UserCmd             # their most recent movement command
  wishdir  : vec3                # the direction derived from it
  message  : SizeBuf             # their RELIABLE stream; appended to at any
                                 # time, flushed once per frame
  edict    : Edict               # always entity (slot index + 1)
  name     : text[32]
  colors   : int                 # packed shirt and trouser colours
  ping_times : real[16]          # a ring of round-trip samples
  num_pings  : int
  spawn_parms : real[16]         # carried between levels
  old_frags   : int              # so a score change can be detected
```

**Invariants** — the split between the two server records is the level-change contract:
everything in the per-level record is discarded and rebuilt, everything in the per-process
record survives. So the episode flags, the player slots and each player's sixteen carry
values persist, and nothing else does.

**A player is always entity slot index plus one.** Player 0 is entity 1. That identity is
relied on throughout ([`pr_edict.c`](pr_edict.c.md#ed_alloc) reserves the low slots,
[`pr_cmds.c`](pr_cmds.c.md) converts back and forth) and it is what lets the wire address a
player by a single number.

The allocated player count never shrinks within a process, because the array is hunk memory
taken before the level's mark. So raising the player limit mid-session needs a restart, and
lowering it wastes the slots.

The **three broadcast buffers** are the delivery model: unreliable for this frame's
ephemeral state, reliable for changes a client must not miss, and the signon for everything
a client needs on arrival. Each has different lifetime and each is flushed differently
([`sv_main.c`](sv_main.c.md)).

The **model index is the position in the precache list**, and index 0 is always the map
itself. That is why the map's asset path is stored separately.

## Movement types

```text
CONSTANT movetype_none         = 0   # never moves; no physics at all
CONSTANT movetype_anglenoclip  = 1   # angles change; position does not; no
CONSTANT movetype_angleclip    = 2   # collision / with collision
CONSTANT movetype_walk         = 3   # a player: gravity, stepping, friction
CONSTANT movetype_step         = 4   # a monster: gravity, but moves in
                                     # discrete steps and sits on ledges
CONSTANT movetype_fly          = 5   # free movement, no gravity
CONSTANT movetype_toss         = 6   # gravity, stops dead on impact
CONSTANT movetype_push         = 7   # map geometry that moves: does not clip
                                     # against the world, and PUSHES or CRUSHES
                                     # what is in its way
CONSTANT movetype_noclip       = 8   # moves through everything
CONSTANT movetype_flymissile   = 9   # no gravity, and enlarged against monsters
CONSTANT movetype_bounce       = 10  # gravity, bounces on impact
```

**Invariants** — a closed set, each with a branch in [`sv_phys.c`](sv_phys.c.md). The three
worth naming:

**Stepped** movement is not walking with a different constant. A monster moves in discrete
displacements decided by the game logic, and the engine's job is to refuse a step that would
walk it off a ledge ([`sv_move.c`](sv_move.c.md)). That refusal is why monsters do not fall
off walkways and players do.

**Pushing** movement inverts the normal relationship: the entity is map geometry, it does not
collide against the world, and anything in its path is either displaced or killed. Doors,
platforms and trains are all this. It is also the only movement type whose entity may be
solid map geometry ([`world.c`](world.c.md#sv_hullforentity) enforces the pairing).

**Flying missile** differs from plain flying only in the enlarged collision box against
monsters, which is the missile trace mode ([`world.h`](world.h.md#trace-modes)).

Two more types exist behind the sequel's build switch: a bouncing projectile without gravity,
and one that simply tracks whatever it is attached to.

## Solidity

```text
CONSTANT solid_not      = 0   # no interaction at all
CONSTANT solid_trigger  = 1   # detects overlap; does not block
CONSTANT solid_bbox      = 2  # a box that blocks, and that can be stood on
CONSTANT solid_slidebox  = 3  # a box that blocks, but cannot be stood on
CONSTANT solid_bsp       = 4  # map geometry: exact collision, blocks
```

**Invariants** — triggers live in a separate list per spatial node from solid entities
([`world.c`](world.c.md#sv_linkedict)), so the two are never confused and a sweep never
consults triggers at all.

The distinction between the two blocking boxes is whether a player landing on it counts as
on the ground. Monsters use the sliding form so that a player cannot ride a monster's head;
crates use the plain form so that a player can stand on them.

## Entity flags

```text
CONSTANT fl_fly            = 1     # movement ignores gravity
CONSTANT fl_swim           = 2
CONSTANT fl_conveyor       = 4
CONSTANT fl_client         = 8     # this entity is a player
CONSTANT fl_inwater        = 16    # engine-maintained
CONSTANT fl_monster        = 32    # subject to the enlarged missile box
CONSTANT fl_godmode        = 64
CONSTANT fl_notarget       = 128   # monsters ignore this entity
CONSTANT fl_item           = 256   # bounding box expanded 15 units
                                   # horizontally when linked
CONSTANT fl_onground       = 512   # engine-maintained
CONSTANT fl_partialground  = 1024  # standing on an edge: not all corners
                                   # supported
CONSTANT fl_waterjump      = 2048  # climbing out of water; physics suspended
CONSTANT fl_jumpreleased   = 4096  # jump debouncing: the key must be released
                                   # between jumps
```

**Invariants** — four of these are set by the engine and read by the game: in water, on
ground, partially on ground, and the water-jump state. The rest are set by the game and read
by the engine. That two-way traffic through one field is why the field is shared
([`progdefs.q1`](progdefs.q1.md)) rather than living on either side.

The **item** flag reaches into the collision system's bounding-box expansion, and the
**monster** flag reaches into the missile trace mode. Both are cases where a game-set flag
changes engine physics.

**Jump debouncing** is a flag rather than an edge detection because the movement command
carries button *state*, not transitions ([`protocol.h`](protocol.h.md)), so "has the jump
key been released since the last jump" has to be remembered somewhere.

## Entity effects

```text
CONSTANT ef_brightfield = 1   # a field of sparkling particles
CONSTANT ef_muzzleflash = 2   # a one-frame light at the entity's front
CONSTANT ef_brightlight = 4   # a large dynamic light
CONSTANT ef_dimlight    = 8   # a small one
```

**Invariants** — set by the game, transmitted in the entity state
([`quakedef.h`](quakedef.h.md)), and interpreted entirely on the **client**
([`cl_main.c`](cl_main.c.md)). So these four bits are a contract between the game logic and
the client, with the server only forwarding them.

## Spawn flags

```text
CONSTANT spawnflag_not_easy        = 256
CONSTANT spawnflag_not_medium      = 512
CONSTANT spawnflag_not_hard        = 1024
CONSTANT spawnflag_not_deathmatch  = 2048
```

**Invariants** — set by the map author, read by the engine's level loader
([`pr_edict.c`](pr_edict.c.md#ed_loadfromfile)) to decide which entities exist at all. The
low eight bits of the same field belong to the game logic, which is why these start at 256.

## Damage and death

```text
CONSTANT dead_no = 0 ;  dead_dying = 1 ;  dead_dead = 2
CONSTANT damage_no = 0 ;  damage_yes = 1 ;  damage_aim = 2
```

**Invariants** — the third damage setting means "and the auto-aim may target this", which is
what [`pr_cmds.c`](pr_cmds.c.md#pf_aim-aimentity-float---vector-44) tests. So the game decides what auto-aim helps with
by setting a damage mode, not by a separate flag.

## Globals and entry points

```text
VARIABLE svs : ServerStatic ;  sv : Server
VARIABLE host_client : Client      # whose command is being processed
VARIABLE sv_player   : Edict       # that client's entity
VARIABLE host_abortserver : jump target    # the error unwind destination
VARIABLE teamplay, skill, deathmatch, coop, fraglimit, timelimit : Cvar
```

**Invariants** — the current client is a **global**, set around every per-player operation.
Every command handler that acts on "the player who sent this" reads it
([`cmd.h`](cmd.h.md)), and it is the reason such handlers take no arguments.

The abort target is a non-local jump used by the error path
([`host.c`](host.c.md#host_error)) to abandon a frame from arbitrary depth, including from
inside the interpreter. A rebuild with exceptions uses one; a rebuild without needs the same
unwind and must reproduce the state the handlers clean up.

**Contract** — the declared entry points divide as: initialization; the level loader; the
per-frame client loop and physics; the three message-composing functions; the
message-sending function; two per-player print functions; two navigation helpers shared
with the host-function table; and the carry-value save. Each is documented in its
implementing file: [`sv_main.c`](sv_main.c.md), [`sv_phys.c`](sv_phys.c.md),
[`sv_move.c`](sv_move.c.md), [`sv_user.c`](sv_user.c.md).
