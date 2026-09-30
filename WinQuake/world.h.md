# WinQuake/world.h

> The collision interface: a trace result record, three trace modes, and the five operations everything that moves goes through.

**Needs** — [`mathlib.h`](mathlib.h.md) · [`progs.h`](progs.h.md) (the entity type)
**Used by** — [`world.c`](world.c.md) · [`sv_phys.c`](sv_phys.c.md) · [`sv_move.c`](sv_move.c.md) · [`sv_user.c`](sv_user.c.md) · [`pr_cmds.c`](pr_cmds.c.md) · [`quakedef.h`](quakedef.h.md) includes it for everyone
**Tier floor** — none

## Purpose

Every question about "can this go there" is answered by one function in
[`world.c`](world.c.md), and this file is its signature. The record it returns is the
engine's collision vocabulary and is also — copied field by field — what game logic sees
after a trace ([`pr_cmds.c`](pr_cmds.c.md#pf_traceline-tracelinevector-vector-float-entity-16)).

## State

```text
RECORD Plane                # a plane, in the collision system's own form
  normal : vec3
  dist   : real

RECORD Trace                # the result of a swept-box query
  allsolid   : bool         # the whole sweep was inside solid; the plane is
                            # then meaningless
  startsolid : bool         # the sweep BEGAN inside solid
  inopen     : bool         # the sweep passed through empty space
  inwater    : bool         # the sweep passed through a liquid
  fraction   : real         # how far along the sweep it got; 1 means it
                            # completed
  endpos     : vec3         # where it stopped
  plane      : Plane        # the surface it hit, oriented toward the mover
  ent        : optional<Edict>   # what it hit; nothing when it hit the world
                                 # geometry or nothing at all
```

**Invariants** — the four flags are not mutually exclusive and each answers a different
question. `fraction` of 1 with `allsolid` set means the sweep never left solid; `fraction` of
1 with `allsolid` clear means a clean move. `startsolid` set means the mover was already
stuck, and physics then has to extract it rather than stop it. `inopen` and `inwater` are
what tell game logic whether a line of sight crossed a water surface, which is how the
game decides a shot fired from air into water does not reach.

The hit entity being *nothing* for world geometry is why every consumer substitutes the
world entity ([`pr_cmds.c`](pr_cmds.c.md#pf_traceline-tracelinevector-vector-float-entity-16)).

## Trace modes

```text
CONSTANT move_normal     = 0   # collide with everything solid
CONSTANT move_nomonsters = 1   # ignore anything that is not map geometry —
                               # used for line of sight and for edge tests
CONSTANT move_missile    = 2   # against MONSTERS ONLY, enlarge the moving box
                               # to 30 units on a side
```

**Invariants** — the missile mode is the interesting one. A rocket is a point, and a point
passing a monster in one frame's step could miss it entirely; so against monsters
specifically, the moving object is inflated to a 30-unit cube while staying its real size
against everything else. That is why rockets in this game feel generous, and it is a
deliberate gameplay decision rather than a fix for tunnelling — geometry still uses the
real size.

## `SV_ClearWorld`

**Contract** — builds the spatial index. Must be called after the map is loaded and before
any entity is linked.

## `SV_UnlinkEdict`

**Contract** — removes an entity from the spatial index. Must be called before removing an
entity, and before moving one, so that it does not collide with its own old position.

## `SV_LinkEdict`

**Contract** — inserts or reinserts an entity into the spatial index, recomputing its
world-space bounding box and its list of touched map leaves. Must be called after any change
to an entity's position, bounding box or solidity. When asked, also runs the touch callbacks
of every trigger the entity now overlaps.

**Invariants** — the world-space box and the leaf list are *outputs* of this call, and
nothing else may write them. Game logic reading them
([`progdefs.q1`](progdefs.q1.md)) is reading engine-maintained state.

## `SV_PointContents`, `SV_TruePointContents`

**Contract** — return the world's contents value at a point, ignoring all entities. The
plain form collapses the six directional-current contents into plain water; the true form
does not.

**Invariants** — the two exist because most code wants "is this water" while the physics
that applies a current needs to know which way. A rebuild that has only one of them gets
either no currents or wrong liquid damage.

## `SV_TestEntityPosition`

**Contract** — takes an entity; reports whether it is currently stuck inside something.

## `SV_Move`

**Contract** — sweeps a box from one point to another and returns what it hit. The box is
given as a minimum and maximum **relative to the start point**. One entity may be excluded.
The mode selects what counts as solid.

**Invariants** — three behaviours the header states explicitly and a rebuild must
reproduce:

- A sweep entirely inside solid sets `allsolid`.
- A sweep **starting** inside solid is allowed to continue *out* into open space rather than
  stopping immediately. That is what lets a stuck entity free itself.
- The excluded entity never collides, and neither does anything whose owner is the excluded
  entity, nor the excluded entity's own owner — see
  [`world.c`](world.c.md#sv_cliptolinks). A rocket does not hit the player who fired it.
