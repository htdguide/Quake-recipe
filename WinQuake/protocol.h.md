# WinQuake/protocol.h

> The game protocol: version 15, thirty-five server messages, five client messages, and the bit-flag scheme that keeps an entity update down to a handful of bytes.

**Needs** — nothing; this is a leaf
**Used by** — [`sv_main.c`](sv_main.c.md) and [`sv_user.c`](sv_user.c.md) (the writers) · [`cl_parse.c`](cl_parse.c.md) and [`cl_tent.c`](cl_tent.c.md) (the readers) · [`host_cmd.c`](host_cmd.c.md) · [`quakedef.h`](quakedef.h.md) includes it for everyone
**Tier floor** — none; a wire layout

## Purpose

This is the contract between a server and a client of *different builds*, and it is the
single most compatibility-critical file in the engine. It is worth reading alongside
[`cl_parse.c`](cl_parse.c.md), which decodes it, and
[`sv_main.c`](sv_main.c.md), which produces it.

Two design decisions carry the file. First, **there is no delta compression**: the
server sends every visible entity's changed fields every frame, and "changed" means
"differs from the entity's spawn baseline", not "differs from what this client last
received". Second, **the flag word is the message identity**: an entity update's tag
byte has its high bit set, and the remaining seven bits are the first seven flags of
the update itself, with an eighth flag asking for a second flag byte. So the common
case — an entity that moved a little — is a tag byte, a flag byte, and two or three
coordinates.

The absence of per-client delta state is what makes this protocol unusable over the
internet and is exactly what [`QW`](../QW/client/protocol.h.md) replaces.

## State

A wire format. No runtime state.

## Version

```text
CONSTANT protocol_version = 15
```

**Invariants** — sent as the first field of the server-information message and checked
by the client, which refuses a mismatch. Also recorded in every demo file, so a demo is
tied to a protocol version.

## Entity update flags

The tag byte for an entity update has its high bit set. The low seven bits are flags;
the remaining seven flags follow in a second byte when the first flag asks for it.

```text
# In the tag byte itself:
CONSTANT u_morebits = 1 SHIFTED LEFT 0    # a second flag byte follows
CONSTANT u_origin1  = 1 SHIFTED LEFT 1    # x coordinate follows
CONSTANT u_origin2  = 1 SHIFTED LEFT 2    # y
CONSTANT u_origin3  = 1 SHIFTED LEFT 3    # z
CONSTANT u_angle2   = 1 SHIFTED LEFT 4    # yaw follows
CONSTANT u_nolerp   = 1 SHIFTED LEFT 5    # do not interpolate toward this
                                          # position; it teleported
CONSTANT u_frame    = 1 SHIFTED LEFT 6    # animation frame follows
CONSTANT u_signal   = 1 SHIFTED LEFT 7    # the high bit: this byte IS an
                                          # entity update

# In the second flag byte:
CONSTANT u_angle1     = 1 SHIFTED LEFT 8   # pitch
CONSTANT u_angle3     = 1 SHIFTED LEFT 9   # roll
CONSTANT u_model      = 1 SHIFTED LEFT 10
CONSTANT u_colormap   = 1 SHIFTED LEFT 11
CONSTANT u_skin       = 1 SHIFTED LEFT 12
CONSTANT u_effects    = 1 SHIFTED LEFT 13
CONSTANT u_longentity = 1 SHIFTED LEFT 14  # the entity number is a short,
                                           # not a byte
```

**Invariants** — the field order on the wire is the **bit order**, low to high, and the
reader must consume them in exactly that order. Yaw is in the first byte and pitch and
roll are in the second, because yaw is the angle that changes on almost every entity
and the other two rarely change at all. That asymmetry is a deliberate byte saving and
a rebuild must not normalize it.

The teleport flag is not cosmetic: without it the client interpolates an entity across
the map over one frame, drawing a streak. The server sets it when the origin changed by
more than a threshold.

The entity number is one byte unless the long flag is set, which caps the cheap form at
255 entities against a level limit of 600.

## Player-state flags

The client's own state — health, velocity, view offsets — arrives in one message with
its own sixteen-bit flag word.

```text
CONSTANT su_viewheight  = 1 SHIFTED LEFT 0   # a signed byte, world units
CONSTANT su_idealpitch  = 1 SHIFTED LEFT 1   # a signed byte, degrees
CONSTANT su_punch1..3   = 1 SHIFTED LEFT 2..4  # view angle kick, per axis
CONSTANT su_velocity1..3= 1 SHIFTED LEFT 5..7  # a signed byte, in 16ths
# bit 8 is unused — an available bit, as the source notes
CONSTANT su_items       = 1 SHIFTED LEFT 9   # a long: the item bit set
CONSTANT su_onground    = 1 SHIFTED LEFT 10  # NO DATA: the bit is the value
CONSTANT su_inwater     = 1 SHIFTED LEFT 11  # NO DATA: the bit is the value
CONSTANT su_weaponframe = 1 SHIFTED LEFT 12
CONSTANT su_armor       = 1 SHIFTED LEFT 13
CONSTANT su_weapon      = 1 SHIFTED LEFT 14
```

**Invariants** — two of the flags carry no payload: their presence *is* the boolean.
Every other set flag is followed by data. A reader that consumes a byte for those two
desynchronizes the rest of the message.

Velocity is quantized to sixteenths of a unit per component in a signed byte, so the
client's own velocity is known to ±8 units per second and only up to ±2048. It is used
only for the view bob and the fall-damage sound, so the coarseness is acceptable —
which is worth stating, because a rebuild that adds client-side prediction on this
protocol cannot use this field for it.

The health, frags, ammo counts and weapon are *not* in this message: they travel as
numbered statistic updates, which are sent only when they change.

## Sound flags

```text
CONSTANT snd_volume      = 1 SHIFTED LEFT 0   # a byte follows; else 255
CONSTANT snd_attenuation = 1 SHIFTED LEFT 1   # a byte follows; else 1.0
CONSTANT snd_looping     = 1 SHIFTED LEFT 2   # a long follows
```

**Invariants** — the defaults are the common case and are compiled into both ends, so
most sounds cost only their flag byte, their entity-and-channel word, their sound
index and their position. The entity and channel share a 16-bit field: the channel is
the low three bits and the entity number the rest, capping entities at 8191 for sound
purposes.

## Server-to-client messages

Thirty-five tags. The high bit of a tag selects an entity update instead, so the tags
themselves occupy 0 through 34.

| Tag | Name | Payload |
|---|---|---|
| 0 | bad | never sent; receiving it is an error |
| 1 | nop | nothing; keeps a connection alive |
| 2 | disconnect | nothing |
| 3 | updatestat | a statistic index (byte) and a value (long) |
| 4 | version | the protocol version (long) |
| 5 | setview | the entity number the camera follows (short) |
| 6 | sound | the flag scheme above |
| 7 | time | the server's clock (float) — **begins every frame's datagram** |
| 8 | print | a string for the console |
| 9 | stufftext | a string pushed into the client's *command queue* |
| 10 | setangle | an absolute view angle triple |
| 11 | serverinfo | version, the map name, the model list, the sound list |
| 12 | lightstyle | a style index (byte) and its animation string |
| 13 | updatename | a player number and a name |
| 14 | updatefrags | a player number and a score (short) |
| 15 | clientdata | the player-state flag scheme above |
| 16 | stopsound | an entity-and-channel word |
| 17 | updatecolors | a player number and a packed shirt/pants colour byte |
| 18 | particle | a position, a direction, a count and a colour |
| 19 | damage | armour and health taken, and where from |
| 20 | spawnstatic | a full entity state for something that never changes |
| 22 | spawnbaseline | an entity number and its full state |
| 23 | temp_entity | a short-lived effect; see the list below |
| 24 | setpause | a byte |
| 25 | signonnum | a byte: the connection handshake stage |
| 26 | centerprint | a string for the middle of the screen |
| 27 | killedmonster | nothing; the client increments its own counter |
| 28 | foundsecret | nothing; likewise |
| 29 | spawnstaticsound | a position, a sound index, volume and attenuation |
| 30 | intermission | a music string |
| 31 | finale | music and text |
| 32 | cdtrack | a track and a loop track |
| 33 | sellscreen | nothing |
| 34 | cutscene | a string |

**Invariants** — tag 21 is skipped; the source records it as a removed message. The
numbering is therefore not contiguous and a rebuild must not renumber.

The **time** message beginning every frame's unreliable datagram is load-bearing: the
client uses the two most recent server times to interpolate entity positions between
frames, and every other message in a datagram is understood to describe the world at
that time.

The **stufftext** message hands a server arbitrary console input on a client, which is
how a server makes a client start recording, switch levels, or set a variable. It is
also the largest trust boundary in the protocol: a client executes what a server sends.
The engine's only mitigation is that the game directory cannot change at run time
([`common.c`](common.c.md#com_initfilesystem)).

The two counter messages carry no payload because they exist purely so the status bar's
secrets and kills counters animate the instant the event happens rather than at the
next statistic push — which means the client's counters can drift from the server's and
are corrected by the next push.

## Client-to-server messages

```text
CONSTANT clc_bad       = 0
CONSTANT clc_nop       = 1
CONSTANT clc_disconnect= 2
CONSTANT clc_move      = 3    # one movement command
CONSTANT clc_stringcmd = 4    # a console line, for the server to execute
```

**Invariants** — five messages, of which two matter. A client's entire influence on the
world is a movement command and a string. The string command is how every non-movement
action — changing weapon, saying something, using an item — reaches the server, and the
server executes it through its own command dispatcher with the source marked as
coming from a client ([`cmd.h`](cmd.h.md)). That marking is the server's whole
authorization model.

## Temporary-entity events

Fourteen one-shot effects, each with its own small payload: bullet and nail impacts, the
plain and tarbaby explosions, three lightning beam kinds, wizard and knight spikes, the
lava splash, the teleport flash, a coloured explosion, and a generic beam.

**Invariants** — the numbering is shared with the game logic, which triggers these by
number ([`qw-qc/`](../qw-qc/README.md)), and the payloads differ per event, so a reader
must switch on the event before it knows how many bytes to consume. Two more events
exist behind a build switch for the sequel's effects and are not in this build.

## Miscellaneous constants

```text
CONSTANT default_viewheight = 22        # world units above the player's origin
CONSTANT game_coop          = 0        # which end-of-level screen plays
CONSTANT game_deathmatch    = 1
```

## Notes

The file's own comment warns that some of these numbers are mirrored in the game
logic's declarations and in the client's message-name table, and that the three must
agree. That is the real content of this file: it is a numbering shared by three
programs, two of which are not in this directory.
