# WinQuake/cl_parse.c

> The message decoder: turns the server's byte stream into client state, resolving every absent field from that entity's spawn baseline, and builds each player's palette translation.

**Needs** — [`client.h`](client.h.md) · [`protocol.h`](protocol.h.md) · [`common.h`](common.h.md) (the message reader) · [`model.h`](model.h.md) · [`sound.h`](sound.h.md) · [`render.h`](render.h.md) · [`vid.h`](vid.h.md) · [`cdaudio.h`](cdaudio.h.md) · [`screen.h`](screen.h.md) · [`sbar.h`](sbar.h.md) · [`cmd.h`](cmd.h.md) · [`zone.h`](zone.h.md) · [`server.h`](server.h.md) (to detect a local server) · [Seam: Operating system services](../SYSTEM-REQUIREMENTS.md#seam-operating-system-services)
**Used by** — [`cl_main.c`](cl_main.c.md) drives it; [`cl_demo.c`](cl_demo.c.md) feeds it recorded messages
**Tier floor** — none

## Purpose

The other half of [`protocol.h`](protocol.h.md). Two things in it are structural rather than mechanical.

**Every absent field falls back to the entity's baseline**, not to its previous value. That is the whole meaning
of this protocol's delta encoding ([`sv_main.c`](sv_main.c.md#sv_writeentitiestoclient)), and it is why an
entity that stops changing keeps its last sent state rather than reverting.

**A player's colours become a 16-kilobyte palette translation**, remapping two colour ranges at every one of
the 64 shading levels. That is how player colours work in a palettized renderer.

## State

Held in the records the sections below describe; this file owns no other long-lived state.

## `CL_EntityNum`

**Contract** — takes an entity number; grows the client's entity count to cover it, giving each new slot the
default colour map, and returns the slot. A number at or above the maximum is a host error.

**Invariants** — the entity number *is* the array index, so the decoder needs no lookup — which is why the
protocol's numbering and the client's array must have the same bound
([`client.h`](client.h.md)).

## `CL_ParseServerInfo`

**Contract** — wipes the client's world state, checks the protocol version, reads the player limit and allocates
the score table, reads the game type and the level's title, then reads the model and sound lists — touching each
name already cached before loading any, and loading them all. A version or player-limit mismatch prints and
returns without connecting.

```text
FUNCTION cl_parse_server_info()
  cl_clear_state()
  IF the protocol version read does not match ours  print and RETURN
  cl.maxclients = a byte, VALIDATED into 1..16
  cl.scores = hunk-allocate that many score records
  cl.gametype = a byte
  cl.levelname = a string
  print the level's title, framed by a box-drawing rule

  # Read the model names, then the sound names, each terminated by an empty
  # string.
  FOR EACH name  record it
  # TOUCH everything already cached BEFORE loading anything, so that loading
  # one asset does not evict another this level needs.
  FOR EACH model name  mod_touch_model(name)
  FOR EACH sound name  s_touch_sound(name)
  FOR EACH model name  cl.model_precache[i] = mod_for_name(name)
  FOR EACH sound name  cl.sound_precache[i] = s_precache_sound(name)
  cl.worldmodel = cl.model_precache[1]
  r_new_map()
  send keepalives while loading                 # see below
```

**Invariants** — four things.

**Touching before loading is the cache-thrash avoidance** the comment describes: the evictable cache
([`zone.h`](zone.h.md)) orders by recency, so marking everything this level needs *before* loading anything
makes the outgoing level's assets the eviction candidates. Without it, loading the last model can evict the
first.

**The lists are read from index 1**, matching the server's numbering
([`sv_main.c`](sv_main.c.md#sv_sendserverinfo)) whose index 0 is a placeholder.

**The world model is index 1**, not 0.

**The level title is printed framed by a rule of box-drawing glyphs** from the engine's own font, and with the
high-bit highlight byte — which is how the title appears highlighted and boxed
([`draw.h`](draw.h.md#text)).

## `CL_KeepaliveMessage`

**Contract** — called repeatedly during a long load; reads and discards any pending message, verifying that it is
a do-nothing, and sends a do-nothing of its own at most every five seconds. Does nothing when the server is
local or during demo playback. Any real message arriving is a host error.

```text
FUNCTION cl_keepalive_message()
  IF the server is local OR playing back a demo  RETURN
  # The decoder's global buffer is in use by the caller, so SAVE it.
  saved = the incoming message buffer and its contents
  REPEAT
    ret = get the next message
    IF it is a RELIABLE message  FAIL WITH "received a message"
    IF it is unreliable AND not a no-op  FAIL WITH "datagram wasn't a nop"
  UNTIL nothing is waiting
  restore the incoming buffer
  IF less than five seconds since the last keepalive  RETURN
  send a client no-op
```

**Invariants** — **the incoming message buffer is saved and restored**, because this runs *inside* the decoding
of another message ([`CL_ParseServerInfo`](#cl_parseserverinfo) calls it between asset loads) and the decoder's
cursor and buffer are global ([`common.h`](common.h.md)). That save is the price of the reader being a
singleton.

Receiving a real message here is fatal because there is no way to process it mid-load; the server is supposed to
send only keepalives to a client that has not finished the handshake
([`sv_main.c`](sv_main.c.md#sv_sendclientmessages)).

## `CL_ParseUpdate`

**Contract** — decodes one entity update. Completes the handshake if this is the first update. Reads the
optional second flag byte and the entity number, then each flagged field in bit order, **defaulting every
unflagged field to the entity's baseline**. Shifts the previous position and angles down for interpolation.
Forces the next frame to skip interpolation when the entity was absent from the previous message or the update
is flagged as a teleport.

```text
FUNCTION cl_parse_update(bits)
  IF this is the first update  advance the handshake to complete and reply

  IF bits HAS more-bits      bits = bits BITOR (a byte SHIFTED LEFT 8)
  num = a short IF bits HAS long-entity ELSE a byte
  ent = cl_entity_num(num)

  # If this entity was not in the PREVIOUS message, there is nothing to
  # interpolate from.
  forcelink = (ent.msgtime != cl.mtime[1])
  ent.msgtime = cl.mtime[0]

  modnum = a byte IF bits HAS model ELSE ent.baseline.modelindex
  model = cl.model_precache[modnum]
  IF model != ent.model
    ent.model = model
    IF model EXISTS
      # A new model resets the animation phase: randomized or synchronized,
      # by the model's own setting.
      ent.syncbase = a random fraction IF the model is randomized ELSE 0
    ELSE
      forcelink = true              # a null model: see the note

  ent.frame    = a byte IF bits HAS frame    ELSE ent.baseline.frame
  i            = a byte IF bits HAS colormap ELSE ent.baseline.colormap
  ent.colormap = the default shading table IF i == 0
                 ELSE cl.scores[i-1].translations
  ent.skinnum  = a byte IF bits HAS skin     ELSE ent.baseline.skin
  ent.effects  = a byte IF bits HAS effects  ELSE ent.baseline.effects

  # Shift the previous update down, then read the new one.
  ent.msg_origins[1] = ent.msg_origins[0]
  ent.msg_angles[1]  = ent.msg_angles[0]
  FOR EACH axis j, IN BIT ORDER (origin j then angle j)
    ent.msg_origins[0][j] = a coordinate IF flagged ELSE the baseline's
    ent.msg_angles[0][j]  = an angle     IF flagged ELSE the baseline's

  IF bits HAS no-lerp  ent.forcelink = true
  IF forcelink
    # No previous frame: collapse both slots onto the new value.
    ent.msg_origins[1] = ent.msg_origins[0] ;  ent.origin = it
    ent.msg_angles[1]  = ent.msg_angles[0]  ;  ent.angles = it
    ent.forcelink = true
```

**Invariants** — six things.

**The first entity update completes the handshake.** There is no explicit "you are in the game" message; the
arrival of an entity update is it. That is why the handshake's fourth stage is a client-side action only
([`cl_main.c`](cl_main.c.md#cl_signonreply)).

**Every unflagged field defaults to the baseline**, which is what makes an omitted field mean "as it spawned"
rather than "unchanged". A rebuild that defaults to the previous value produces entities that drift.

**The field order is the bit order and the coordinates and angles are interleaved per axis**
([`protocol.h`](protocol.h.md)). A reader that reads all three coordinates then all three angles misparses
every update.

**A colour map of zero means the default shading table**, and any other value indexes the player score table
minus one — so colour map *n* is player *n*, which is what the server's baseline construction guarantees
([`sv_main.c`](sv_main.c.md#sv_createbaseline)).

**A model change resets the animation phase**, randomized or not by the model's own synchronization setting
([`modelgen.h`](modelgen.h.md)). So a torch that changes model re-randomizes.

**A null model forces the interpolation skip**, which the source marks as a hack to make players with no model
work — a player whose model has not loaded would otherwise interpolate from a stale position.

## `CL_ParseBaseline`

**Contract** — reads an entity's spawn state: model index, frame, colour map, skin, and the three coordinates
and angles interleaved.

**Invariants** — this is the state every later update deltas against, and it arrives once, reliably, in the
signon block ([`host_cmd.c`](host_cmd.c.md#host_prespawn_f-prespawn)).

## `CL_ParseClientdata`

**Contract** — decodes the player's own state from its sixteen-bit flag word: view height, ideal pitch, view
kick, velocity, items, ground and water flags, weapon animation frame, armour, weapon model, health and the
five ammunition counts. Unflagged fields take their defaults. Notifies the status bar when any displayed value
changed.

**Invariants** — **two flags carry no payload** ([`protocol.h`](protocol.h.md)) and reading a byte for them
desynchronizes the rest of the message.

The item field is a composite of the item set and either a second set or the episode flags, packed by the server
([`sv_main.c`](sv_main.c.md#sv_writeclientdatatomessage)); the status bar unpacks it
([`sbar.c`](sbar.c.md)).

The status-bar notification is per changed field, which is what the bar's redraw discipline requires
([`sbar.h`](sbar.h.md)).

## `CL_NewTranslation`

**Contract** — builds one player's palette translation: a copy of the whole shading table with two sixteen-entry
colour ranges remapped to that player's shirt and trouser colours, at **every** shading level.

```text
FUNCTION cl_new_translation(slot)
  dest = cl.scores[slot].translations        # 64 levels by 256 entries
  copy the WHOLE shading table into dest
  top    = (the player's colours BITAND 0xF0)        # the shirt row
  bottom = (the player's colours BITAND 0x0F) << 4   # the trouser row
  FOR EACH shading level, advancing both dest and source by 256
    IF top < 128
      copy 16 entries FROM source+top TO dest+the shirt range
    ELSE
      copy them REVERSED                     # "the artists made some
                                             #  backwards ranges. sigh."
    ...the same for the trouser range
```

**Invariants** — three things.

**The translation is the full 64-by-256 shading table, not a 256-entry map.** A palettized renderer shades by
looking up a row of that table ([`vid.h`](vid.h.md)), so a colour substitution must be applied at every level.
That is why a player costs 16 kilobytes of client memory.

**Palette rows at or above 128 are reversed**, and the source's comment explains: some of the artwork's colour
ramps run dark-to-light and others light-to-dark, so half the ranges must be copied backwards. A rebuild must
reproduce the reversal or half the player colours come out inverted.

**The two remapped ranges are at fixed palette offsets** ([`render.h`](render.h.md)), which is a constraint on
the player model's artwork: its shirt and trousers must be painted in exactly those ranges.

## `CL_ParseStatic`

**Contract** — reads a baseline into a static entity slot, copies it to the current state, and adds the entity's
fragments. A count over the limit is a host error.

**Invariants** — a static entity is created once from the signon block, never updated, and never removed
([`pr_cmds.c`](pr_cmds.c.md#pf_makestatic-makestaticentity-69)). Its fragments are added once and stay — which is why it costs no
per-frame work at all.

## `CL_ParseStartSoundPacket`, `CL_ParseStaticSound`

**Contract** — decode a sound: the flag byte, optional volume and attenuation, the packed entity-and-channel
word, the sound index and the position. The static form reads a position, index, volume and attenuation and
starts a permanent looping sound.

**Invariants** — the entity and channel share one sixteen-bit field, the channel in the low three bits
([`sv_main.c`](sv_main.c.md#sv_startsound)). Volume arrives as a byte and is scaled to a fraction; attenuation
arrives scaled by 64.

## `CL_ParseServerMessage`

**Contract** — the dispatch loop: reads tag bytes until the message is exhausted, handling each. A tag with its
high bit set is an entity update whose low bits are the first seven update flags. An unknown tag is a host
error. A read past the end is a host error.

```text
FUNCTION cl_parse_server_message()
  begin reading
  LOOP
    IF the read overran  FAIL WITH "Bad server message"
    cmd = a byte
    IF cmd == -1  BREAK                         # end of message
    IF cmd HAS the high bit set
      cl_parse_update(cmd BITAND 0x7F)          # the low bits ARE the flags
      CONTINUE
    SELECT cmd
      nop                nothing
      time               cl.mtime[1] = cl.mtime[0]
                         cl.mtime[0] = a float        # the interpolation basis
      clientdata         cl_parse_clientdata(a short)
      version            check it
      disconnect         end the game
      print              print it
      centerprint        show it centred
      stufftext          APPEND it to the console command queue
      damage             record it for the view flash
      serverinfo         cl_parse_server_info()
      setangle           set the view angles absolutely
      setview            cl.viewentity = a short
      lightstyle         record a style's animation string
      sound              cl_parse_start_sound_packet()
      stopsound          stop that entity's channel
      updatename         set a player's name ;  notify the status bar
      updatefrags        set a player's score ;  notify the status bar
      updatecolors       set a player's colours ;  cl_new_translation(slot)
      particle           r_parse_particle_effect()
      spawnbaseline      cl_parse_baseline(the named entity)
      spawnstatic        cl_parse_static()
      temp_entity        cl_parse_tent()
      setpause           set the pause, and start or stop the CD music
      signonnum          advance the handshake and reply
      killedmonster      increment the local counter ;  notify the bar
      foundsecret        likewise
      updatestat         set a numbered statistic
      spawnstaticsound   cl_parse_static_sound()
      cdtrack            record the track and start it
      intermission       enter the end-of-level state
      finale             likewise, with text
      sellscreen         show the order screen
      cutscene           show text
      otherwise          FAIL WITH "Illegible server message"
```

**Invariants** — five things.

**The high bit distinguishes an entity update from a tagged message**, and the low seven bits *are* the first
seven update flags ([`protocol.h`](protocol.h.md)). So the common case costs one byte of tag.

**The time message shifts the previous timestamp down**, and those two values are the entire basis of
interpolation ([`cl_main.c`](cl_main.c.md#cl_lerppoint)). Every datagram begins with one.

**Stuffed text is appended to the command queue**, not inserted, so it runs after the current frame's commands.
That is the trust boundary [`protocol.h`](protocol.h.md) describes.

**The two counter messages increment client-side counters** that a later statistic push may correct, which is
why they exist at all — the counters animate immediately.

**An unknown tag is fatal.** There is no length prefix on a message, so the decoder cannot skip what it does not
understand; a single unknown byte desynchronizes everything after it.
