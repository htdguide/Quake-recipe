# WinQuake/quakedef.h

> The one header every source file includes: the engine's global limits, the protocol's shared numbering, and the include order that fixes what is declared before what.

**Needs** — [`common.h`](common.h.md) · [`bspfile.h`](bspfile.h.md) · [`vid.h`](vid.h.md) · [`sys.h`](sys.h.md) · [`zone.h`](zone.h.md) · [`mathlib.h`](mathlib.h.md) · [`wad.h`](wad.h.md) · [`draw.h`](draw.h.md) · [`cvar.h`](cvar.h.md) · [`screen.h`](screen.h.md) · [`net.h`](net.h.md) · [`protocol.h`](protocol.h.md) · [`cmd.h`](cmd.h.md) · [`sbar.h`](sbar.h.md) · [`sound.h`](sound.h.md) · [`render.h`](render.h.md) · [`client.h`](client.h.md) · [`progs.h`](progs.h.md) · [`server.h`](server.h.md) · [`model.h`](model.h.md) and [`d_iface.h`](d_iface.h.md) *or* [`gl_model.h`](gl_model.h.md) and [`glquake.h`](glquake.h.md) · [`input.h`](input.h.md) · [`world.h`](world.h.md) · [`keys.h`](keys.h.md) · [`console.h`](console.h.md) · [`view.h`](view.h.md) · [`menu.h`](menu.h.md) · [`crc.h`](crc.h.md) · [`cdaudio.h`](cdaudio.h.md)
**Used by** — every `.c` file in the engine, as its first include
**Tier floor** — none of its own; it records the tier decisions made elsewhere

## Purpose

Two jobs. It is the engine's configuration record — every fixed limit, every number
shared between the engine and the game logic, every version constant — and it is the
declaration order, which is real information: it says that the byte buffer must be
declared before the network layer, that the model representation depends on which
renderer is being built, and that the host loop depends on everything.

A rebuild does not need a file like this. What it needs from this page is the limit
table, the reason each limit is where it is, and the knowledge that several of these
numbers are also compiled into the game logic and therefore cannot be changed on one
side alone.

## State

Declares the host's parameter record and the handful of globals the host owns.

```text
RECORD QuakeParms          # what the platform backend hands the engine
  basedir  : text          # the directory holding the game directories
  cachedir : optional<text># development mirror; usually nothing
  argc     : int
  argv     : list<text>
  membase  : pointer       # the one memory block
  memsize  : int

VARIABLE host_parms       : QuakeParms
VARIABLE host_initialized : bool     # registration is closed once this is set
VARIABLE host_frametime   : real     # seconds the current frame represents
VARIABLE host_framecount  : int      # frames since start; never reset
VARIABLE realtime         : real     # seconds since start; never reset, unbounded
VARIABLE host_basepal     : bytes    # 256 RGB triples
VARIABLE host_colormap    : bytes    # 256 x 64 palette-index lighting table
VARIABLE current_skill    : int      # the skill the loaded level actually uses
VARIABLE isDedicated      : bool
VARIABLE minimum_memory   : int
```

**Invariants** — `host_frametime` is the simulation's time step and is clamped
elsewhere ([`host.c`](host.c.md)); `realtime` is wall time and is not. The
distinction matters: physics uses the former, timing and timeouts use the latter.

`current_skill` is a snapshot rather than a live read of the skill variable, because
changing the variable mid-level must not change the level's behaviour.

## Limits

Every number below is a fixed capacity. The ones marked *shared* also appear in the
game logic's own declarations ([`qw-qc/defs.qc`](../qw-qc/defs.qc.md)) or in the
protocol, and changing one side alone breaks compatibility.

| Limit | Value | Why this number |
|---|---|---|
| Minimum memory | 0x550000 (≈5.4 MB) | Below this the surface cache cannot hold one screen's worth of textures. A level pack raises it by 1 MB. |
| Command-line arguments | 50 | Plus seven reserved slots for the safe-mode expansion. |
| In-game path length | 64 | *Shared*: it is the width of an archive entry's name and of a model or sound name. |
| Host path length | 128 | Platform paths; not shared. |
| Plane-side epsilon | 0.1 | The distance within which a point counts as *on* a plane. Load-bearing: it is what stops an entity resting exactly on a surface from oscillating between sides. |
| Reliable message | 8000 bytes | *Shared*: the largest signon or reliable payload. A level whose signon data exceeds it cannot be served. |
| Unreliable message | 1024 bytes | *Shared*: one frame's worth of entity updates must fit, or the frame is dropped whole. |
| Entities per level | 600 | *Shared*: the engine's entity array and the wire's entity numbering. The source's own comment is three exclamations of regret. |
| Light styles | 64 | *Shared*: a style index is a byte on the wire but the array is 64 entries. |
| Models per level | 256 | *Shared*: a model index is one byte on the wire, so this cannot be raised without a protocol change. |
| Sounds per level | 256 | *Shared*, same reason. |
| Style string | 64 | The animation string for one light style; also the period in frames. |
| Client stat slots | 32 | *Shared*: the numbered slots below. |
| Players | 16 | *Shared*: the wire numbers players in a byte but the scoreboard is 16. |
| Player name | 32 | *Shared*. |
| Sound channels used by game logic | 8 | *Shared*: the game addresses a sound channel per entity by number. |
| Cache alignment | 32 | The alignment the software renderer's hot structures are padded to; duplicated in the assembly's own header and must agree. |

## Axis names

```text
CONSTANT PITCH = 0     # nose up and down; POSITIVE LOOKS DOWN
CONSTANT YAW   = 1     # turn left and right
CONSTANT ROLL  = 2     # bank
```

**Invariants** — an angle triple is indexed in this order, not in x-y-z order, and the
pitch sign is inverted relative to intuition. Both are fixed by the map format, the
model format and the protocol. See [`mathlib.c`](mathlib.c.md#anglevectors).

## Player statistics

Fifteen numbered slots the server pushes to the client and the status bar reads:
health, frags, current weapon, ammo, armour, weapon animation frame, four ammo
counts, active weapon, total secrets, total monsters, secrets found, monsters killed.

**Invariants** — *shared with the game logic*, which writes them by number. Two of
them — secrets found and monsters killed — are also incremented on the client by
dedicated protocol messages, so the client's copy can advance without a stat update.
That redundancy exists so the counters animate immediately rather than at the next
stat push.

## Item bits

A 32-bit set naming weapons, ammo types, armour tiers, powerups and the four episode
sigils. Two alternative numberings follow it for the two mission packs, which reassign
most of the bits.

**Invariants** — *shared with the game logic*, which is where the bits are actually
set. The three numberings are selected by [`common.c`](common.c.md#com_initargv)'s
game-variant flags and are consumed by [`sbar.c`](sbar.c.md), which is the only place
the difference is visible. A rebuild should model this as three named sets rather than
one, because no bit means the same thing across all three.

The highest four bits are written as shifts because their values exceed the signed
range; one of the mission-pack constants is written as a decimal literal that does not
fit in a signed 32-bit integer at all. A rebuild should treat the whole set as
unsigned.

## Entity state

```text
RECORD EntityState          # the network's view of an entity
  origin     : vec3
  angles     : vec3
  modelindex : int
  frame      : int
  colormap   : int
  skin       : int
  effects    : int
```

**Invariants** — this is the *complete* set of fields the protocol transmits about an
entity. Everything else about an entity — its velocity, its health, what it is
touching — is server-side only. The client's entire visual model of the world is a
few hundred of these, and delta compression in [`QW`](../QW/server/sv_ents.c.md) is
built on comparing them field by field. A rebuild should treat this record as the
protocol boundary it is.

## Build-time switches

Four switches shape the build and are worth naming because they change what a
rebuild must implement.

**Hardware renderer** — selects the hardware model representation and renderer
headers instead of the software ones. Several structures differ between the two
builds, so the two renderers are not interchangeable at run time: they are two
programs sharing most of their source.

**x86 assembly** — derived from the target architecture, this selects the hand-written
inner loops over their portable twins. See
[Seam: Vectorized inner loops](../SYSTEM-REQUIREMENTS.md#seam-vectorized-inner-loops).

**Unaligned access permitted** — derived from the same test. Where set, several
loaders read multi-byte values straight out of a byte buffer.

**Framebuffer locking** — on one platform the framebuffer must be locked around
writes, so every drawing operation is bracketed by a lock and unlock that are empty
elsewhere. That the bracket exists at all is the load-bearing part: a rebuild whose
surface needs locking has somewhere to put it.

## Host entry points

**Contract** — the host loop's surface: initialize from the parameter record, run one
frame given elapsed seconds, shut down, clear memory between levels, run one server
frame, and three ways to fail — a recoverable error that drops to the console, an
end-of-game that disconnects, and a server shutdown. Also a formatted way to push a
command into a client's queue. Implemented in [`host.c`](host.c.md).

## Third-person camera

**Contract** — three entry points and one variable for the chase camera, implemented
in [`chase.c`](chase.c.md). Declared here rather than in a header of its own because
the file is fifty lines long.
