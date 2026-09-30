# WinQuake/client.h

> The client's two state records — one surviving reconnections, one wiped at every level — the movement command, and the fixed pools every drawable thing comes from.

**Needs** — [`render.h`](render.h.md) (the entity and fragment types) · [`model.h`](model.h.md) · [`sound.h`](sound.h.md) · [`net.h`](net.h.md) · [`vid.h`](vid.h.md) · [`common.h`](common.h.md) · [`cvar.h`](cvar.h.md) · [`quakedef.h`](quakedef.h.md)
**Used by** — every `cl_*.c` file · [`view.c`](view.c.md) · [`screen.c`](screen.c.md) · [`sbar.c`](sbar.c.md) · [`menu.c`](menu.c.md) · [`r_main.c`](r_main.c.md) · [`host.c`](host.c.md) · [`host_cmd.c`](host_cmd.c.md) · [`cmd.c`](cmd.c.md) · [`sv_user.c`](sv_user.c.md) (for the pause condition)
**Tier floor** — none

## Purpose

The mirror image of [`server.h`](server.h.md): two records, one per process and one per connection,
with the split marking exactly what survives a level change. And, like the server's, the more
interesting half is the **per-connection** record — because it is the client's entire model of the
world, and everything in it either arrived over the wire or was interpolated from two things that did.

Three ideas carry the file.

**Almost everything is double-buffered for interpolation.** Time, view angles, velocity and every
entity's position and angles are stored as the last two received values plus a derived one. The client
draws at its own frame rate between two server ticks.

**Everything drawable comes from a fixed pool.** Entities, static entities, temporary entities,
lights, beams and entity fragments all have hard maxima and no growth path. The renderer's visible
list is likewise capped at 256.

**The colour shifts are a four-layer tint stack**, which is how damage flashes, item pickups, powerups
and being underwater all tint the screen without interfering.

## State

The records and constants this header declares are the state; they are described in the sections below.

## The movement command

```text
RECORD UserCmd
  viewangles : vec3             # where the player is looking
  forwardmove, sidemove, upmove : real    # intended velocity, units per second
```

**Invariants** — **the command carries intended velocities, not key states.** The client has already
resolved key presses, speed modifiers and mouse motion into three signed speeds. So the server never
sees a key, and the client's input configuration is entirely its own business
([`cl_input.c`](cl_input.c.md)). That is a genuinely good boundary and a rebuild should keep it.

The button states and the impulse travel *separately* in the message
([`sv_user.c`](sv_user.c.md#sv_readclientmove)), not in this record — which is why the record has no
fields for them.

## Display records

```text
RECORD LightStyle
  length : int
  map    : text[64]             # letters 'a' (dark) to 'z' (bright), cycled

RECORD Scoreboard               # one player, as this client sees them
  name         : text[32]
  entertime    : real
  frags        : int
  colors       : int            # two 4-bit fields: shirt and trousers
  translations : byte[64*256]   # the palette translation, ONE PER SHADE LEVEL

RECORD CShift                   # one layer of the screen tint
  destcolor : int[3]            # the colour to move toward
  percent   : int               # 0 to 256; how far

CONSTANT cshift_contents = 0    # what the player is standing in
CONSTANT cshift_damage   = 1
CONSTANT cshift_bonus    = 2
CONSTANT cshift_powerup  = 3
CONSTANT num_cshifts     = 4
```

**Invariants** — a player's palette translation is **the full shading table remapped**, 64 levels by
256 entries, not a 256-entry map. That is 16 kilobytes per player, per client, and it is why the
scoreboard array is allocated to the actual player count rather than the maximum. The reason is that
the shirt and trouser colour ranges must be remapped *at every light level*
([`cl_parse.c`](cl_parse.c.md#cl_newtranslation)).

The four tint layers are **composited**, not exclusive: taking damage while underwater while holding
a powerup shows all three. Each is a destination colour and a percentage, and
[`view.c`](view.c.md#v_updatepalette) blends them in order.

A light style's animation string is one letter per tenth of a second, cycled — and the string's
*length* is its period. A rebuild must decode letters as brightness, with `a` dark and `z` fully
bright ([`r_light.c`](r_light.c.md#r_animatelight)).

## Per-frame entities

```text
CONSTANT max_dlights = 32
RECORD DLight                   # a dynamic light
  origin   : vec3
  radius   : real
  die      : real               # the time it stops
  decay    : real               # radius lost per second
  minlight : real               # stop contributing below this
  key      : int                # the entity it belongs to, so it can be replaced

CONSTANT max_beams = 24
RECORD Beam                     # a lightning bolt
  entity  : int                 # whose bolt it is
  model   : Model
  endtime : real
  start, end : vec3

CONSTANT max_efrags          = 640
CONSTANT max_temp_entities   = 64     # lightning bolts and the like
CONSTANT max_static_entities = 128    # torches and the like
CONSTANT max_visedicts       = 256    # what one frame may draw
CONSTANT signons             = 4      # handshake messages before playing
```

**Invariants** — a dynamic light carries a **key** naming the entity it belongs to, so that a
muzzle flash from the same weapon replaces the previous one rather than adding a second
([`cl_main.c`](cl_main.c.md#cl_allocdlight)). Without it, rapid fire would exhaust the 32 slots.

Thirty-two dynamic lights, and the surface's dynamic-light bit set
([`model.h`](model.h.md)) is a 32-bit integer — so the two limits are the same limit and cannot be
raised independently.

**The visible-entity list caps at 256 per frame.** Exceeding it drops entities silently.

## Connection state — survives reconnection

```text
RECORD ClientStatic
  state       : enum { dedicated, disconnected, connected }
  mapstring   : text[64]        # the command that started this level
  spawnparms  : text[2048]      # its arguments, so a restart can repeat it
  demonum     : int             # -1 means the demo loop is off
  demos       : text[8][16]     # the attract-mode playlist
  demorecording, demoplayback, timedemo : bool
  forcetrack  : int             # -1 uses the level's own music track
  demofile    : file
  td_lastframe, td_startframe : int    # the benchmark's metering
  td_starttime : real
  signon      : int             # 0 through 4
  netcon      : QSocket
  message     : SizeBuf         # what to send the server
```

**Invariants** — the three-state connection value distinguishes a **dedicated server** — which can
never have a client — from a client that merely has no connection. That distinction is checked
throughout, because a dedicated server must not try to draw or play sound.

Demo recording state lives here rather than in the per-level record, and the comment says why:
recording is started *before* a level is entered, so it must survive the wipe.

The handshake count of 4 is the four stages
([`host_cmd.c`](host_cmd.c.md#the-connection-handshake)), and `signon` reaching it means "fully in the
game" — a test made all over the client.

## World state — wiped at every connection

```text
RECORD ClientState
  movemessages : int            # commands sent this connection; the first few
                                # are discarded
  cmd          : UserCmd        # the last command sent

  # --- what the status bar shows ---
  stats        : int[32]        # the numbered statistics of quakedef.h
  items        : int            # the item bit set
  item_gettime : real[32]       # when each was acquired, so it can blink
  faceanimtime : real           # how long the pain face stays

  # --- the screen tint ---
  cshifts, prev_cshifts : CShift[4]

  # --- view, all interpolated ---
  mviewangles  : vec3[2]        # the last two received, newest first
  viewangles   : vec3           # the client's OWN angles; see the note
  mvelocity    : vec3[2]        # the last two received
  velocity     : vec3           # interpolated; drives the lean and bob
  punchangle   : vec3           # a temporary offset the server sets

  # --- automatic pitch centring ---
  idealpitch, pitchvel : real
  nodrift    : bool
  driftmove  : real
  laststop   : real

  viewheight : real
  crouch     : real             # a local smoothing of step-ups

  paused, onground, inwater : bool
  intermission   : int          # the end-of-level screen
  completed_time : int          # latched when it began

  # --- time, the interpolation basis ---
  mtime : real[2]               # the last two server timestamps
  time  : real                  # this client's view of now, BETWEEN them
  oldtime : real                # the previous frame's, for decay rates
  last_received_message : real  # for the network-trouble indicator

  # --- static for the connection ---
  model_precache : Model[256]
  sound_precache : Sfx[256]
  levelname   : text[40]
  viewentity  : int             # which entity the camera follows
  maxclients  : int
  gametype    : int

  worldmodel  : Model
  free_efrags : EFrag           # the fragment free list
  num_entities, num_statics : int
  viewent     : Entity          # the weapon model in the player's hands
  cdtrack, looptrack : int
  scores      : list<Scoreboard>    # allocated to maxclients
```

**Invariants** — five things here define how the client behaves.

**The client owns its view angles.** The comment is explicit: the client maintains its own idea and
sends it to the server every frame; the server sets the punch offset and can command an absolute
angle at a level start or after a teleport. So aiming is entirely client-side and has no latency,
which is the single most important feel decision in the client. A rebuild that makes the server
authoritative over aim changes the game fundamentally.

**Time is interpolated between the last two server timestamps.** The client's `time` sits between
them, and every entity's drawn position is the blend at that fraction
([`cl_main.c`](cl_main.c.md#cl_relinkentities)). That is the whole smoothing mechanism, and it means
the client is always displaying the world slightly in the past.

**View angles are *also* double-buffered**, but only for demo playback — during live play the client's
own angles are used directly, and during playback the two received values are blended because there is
no local input.

**The first few movement commands are discarded**, counted by `movemessages`, so that a key held
during a level load does not fire a weapon on the first frame.

**The crouch value is purely local**: it smooths the view's rise when a player steps up, and the
server knows nothing about it ([`view.c`](view.c.md#v_calcrefdef)).

The precache arrays are the client's half of the protocol's model and sound numbering
([`sv_main.c`](sv_main.c.md#sv_sendserverinfo)), built from the names in the server-information
message, and they must agree index for index.

## The global pools

```text
VARIABLE cls : ClientStatic ;  cl : ClientState
VARIABLE cl_efrags          : EFrag[640]
VARIABLE cl_entities        : Entity[600]
VARIABLE cl_static_entities : Entity[128]
VARIABLE cl_lightstyle      : LightStyle[64]
VARIABLE cl_dlights         : DLight[32]
VARIABLE cl_temp_entities   : Entity[64]
VARIABLE cl_beams           : Beam[24]
VARIABLE cl_visedicts       : Entity[256] ;  cl_numvisedicts : int
```

**Invariants** — statically allocated, every one, and the source's own comment marks the fragment pool
as wanting dynamic allocation. The entity array is sized to the server's entity maximum so that a
server entity number indexes it directly — the protocol's entity number *is* the array index, which is
what makes the update decoder a single lookup.

## Input state

```text
RECORD KButton                  # one logical input button
  down  : int[2]                # up to TWO keys currently holding it
  state : int                   # low bit: currently down; see cl_input.c
```

**Invariants** — a button remembers **two** holding keys, so that pressing the same action on two
different keys and releasing one does not release the action. That is why the record is not a boolean,
and it is the source of the classic "stuck movement" bug when a third key is used.

## Tunable variables

**Contract** — the twenty declared variables are the client's entire input configuration: the four
movement speeds, the speed and angle modifier keys, two turn rates, auto-fire, the network trace, the
interpolation disable, the pitch-centring speed, look-spring and look-strafe, mouse sensitivity, and
four mouse axis scalings.

**Invariants** — mouse motion is scaled **per axis by four separate factors**, which is what allows
the mouse to be configured for inverted pitch, for strafing instead of turning, or for moving instead
of pitching. All four are read in one place
([`in_win.c`](in_win.c.md) and its siblings, through
[`cl_input.c`](cl_input.c.md)).

## Entry points

**Contract** — grouped by implementing file: the connection lifecycle and light management
([`cl_main.c`](cl_main.c.md)), the four handshake replies, input gathering and command sending
([`cl_input.c`](cl_input.c.md)), demo recording and playback
([`cl_demo.c`](cl_demo.c.md)), message decoding and the player palette translation
([`cl_parse.c`](cl_parse.c.md)), the view and its palette
([`view.c`](view.c.md)), and temporary entities
([`cl_tent.c`](cl_tent.c.md)).

**Invariants** — the **four separate handshake reply functions** correspond to the four signon stages,
and the client's reply to each is a different console command
([`host_cmd.c`](host_cmd.c.md#the-connection-handshake)). They are declared here and dispatched by
stage number, which is the client's half of the handshake.
