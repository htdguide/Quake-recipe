# WinQuake/snd_dma.c

> The sound system's control half: channel allocation with a priority rule, spatialization, the ambient levels read from the map, and the mixing clock derived from the device's own read position.

**Needs** — [`sound.h`](sound.h.md) · [`client.h`](client.h.md) · [`model.h`](model.h.md) (the ambient levels) · [`bspfile.h`](bspfile.h.md) · [`console.h`](console.h.md) · [`cmd.h`](cmd.h.md) · [`cvar.h`](cvar.h.md) · [`zone.h`](zone.h.md) · [`common.h`](common.h.md) · [Seam: Audio output](../SYSTEM-REQUIREMENTS.md#seam-audio-output)
**Used by** — [`host.c`](host.c.md) initializes and updates it; [`cl_parse.c`](cl_parse.c.md) and [`sv_main.c`](sv_main.c.md) start sounds; [`r_edge.c`](r_edge.c.md) and [`r_main.c`](r_main.c.md) call the mid-frame top-up
**Tier floor** — T2: the mixing clock is derived from a position the hardware advances concurrently

## Purpose

Three things carry this file.

**The mixing clock.** There is no audio thread and no timer. The device reports how many samples it has consumed;
the engine counts wraps, derives an absolute sample position, and mixes forward from there to a target a tenth of
a second ahead. Everything about the sound system's timing follows from that.

**The channel priority rule.** Eight dynamic channels serve every entity in the world, and the rule for which one
a new sound takes is a small piece of game design: a sound on the same entity and channel always wins, a
monster's sound never displaces the player's, and otherwise the channel finishing soonest is taken.

**Ambient levels from the map.** The four ambient channels' volumes come from the leaf the listener is in
([`bspfile.h`](bspfile.h.md)) and fade toward their target, which is what makes approaching water audible before
it is visible.

## State

```text
CONSTANT max_sfx = 512
VARIABLE channels : Channel[128] ;  total_channels : int
VARIABLE known_sfx : Sfx[512] ;  num_sfx : int
VARIABLE ambient_sfx : Sfx[4]
VARIABLE soundtime   : int          # the device's absolute position, in sample
                                    # PAIRS
VARIABLE paintedtime : int          # how far the mixer has written
VARIABLE shm : Dma                  # the device
VARIABLE listener_origin, listener_forward, listener_right, listener_up : vec3
VARIABLE sound_nominal_clip_dist : real = 1000
VARIABLE desired_speed = 11025 ;  desired_bits = 16
VARIABLE snd_blocked : int ;  snd_ambient : bool ;  snd_initialized : bool
VARIABLE volume : Cvar = 0.7 ;  bgmvolume = 1 ;  ambient_level = 0.3
VARIABLE ambient_fade : Cvar = 100 ;  _snd_mixahead : Cvar = 0.1
VARIABLE loadas8bit, nosound, precache, snd_show, snd_noextraupdate : Cvar
VARIABLE fakedma : bool ;  fakedma_updates = 15
```

**Invariants** — both clocks count **sample pairs**, not bytes and not mono samples, so a stereo device's byte
position must be divided by its channel count. Mixing all the way to the target and no further is what bounds
latency at the mix-ahead setting — a tenth of a second by default, which is the sound system's whole latency
budget.

## `S_Init`, `S_Startup`, `S_Shutdown`

**Contract** — register the variables and commands, allocate the sound table, open the device, build the volume
scale table, and load the four ambient sounds. Startup opens the device and records what it negotiated; shutdown
closes it.

**Invariants** — the engine reads back the rate, width and channel count the device actually gave
([`sound.h`](sound.h.md)), rather than assuming what it asked for.

## `S_FindName`, `S_PrecacheSound`, `S_TouchSound`, `S_ClearPrecache`

**Contract** — find or register a sound by name; register it and, when precaching is enabled, load it
immediately; mark a resident one recently used; and forget the registrations.

**Invariants** — touching before loading is the same cache-thrash avoidance the model loader uses
([`cl_parse.c`](cl_parse.c.md#cl_parseserverinfo)).

## `SND_PickChannel`

**Contract** — takes an entity and a channel number; returns the dynamic channel to use, or nothing if none can
be taken. Applies the priority rule and silences whatever occupied the chosen slot.

```text
FUNCTION snd_pick_channel(entnum, entchannel) -> optional<Channel>
  first_to_die = none ;  life_left = a very large value
  FOR EACH dynamic channel
    # 1. A non-zero channel on the same entity ALWAYS wins.
    IF entchannel != 0 AND the channel's entity IS entnum
       AND (its channel IS entchannel OR entchannel IS -1)
      first_to_die = this one ;  BREAK
    # 2. A monster's sound never displaces the PLAYER's.
    IF this channel belongs to the view entity AND entnum is not the view entity
       AND it is playing
      CONTINUE
    # 3. Otherwise take whichever finishes soonest.
    IF its remaining time < life_left
      life_left = it ;  first_to_die = this one
  IF none was chosen  RETURN nothing
  silence the chosen channel ;  RETURN it
```

**Invariants** — three decisions, each audible.

**Channel zero never overrides.** A sound played on channel zero always takes a fresh slot, which is why the game
uses it for one-shot effects and non-zero channels for an entity's voice, weapon and footsteps
([`sound.h`](sound.h.md)).

**The player's own sounds are protected** from being displaced by any other entity's. So in a crowded firefight
the player still hears their own weapon — which is the single most important sound in the game.

**Otherwise the channel finishing soonest is taken**, which is a crude priority: a long sound outlives a short
one. There is no explicit priority field.

A channel number of −1 means "any channel of this entity", used by the stop-sound path.

## `SND_Spatialize`

**Contract** — takes a channel; sets its two ear volumes from its position relative to the listener, its
attenuation, and the device's channel count. A sound belonging to the listener's own entity is full volume in
both ears.

```text
FUNCTION snd_spatialize(ch)
  IF ch.entnum IS the view entity
    ch.leftvol = ch.rightvol = ch.master_vol ;  RETURN     # always full volume
  source_vec = ch.origin - listener_origin
  dist = normalize(source_vec) * ch.dist_mult       # distance, scaled by the
                                                    # sound's attenuation
  dot = dot(listener_right, source_vec)             # -1 fully left, +1 fully
                                                    # right
  IF the device is mono  lscale = rscale = 1
  ELSE                   rscale = 1 + dot ;  lscale = 1 - dot
  ch.rightvol = max(ch.master_vol * (1 - dist) * rscale, 0)
  ch.leftvol  = max(ch.master_vol * (1 - dist) * lscale, 0)
```

**Invariants** — four things.

**The player's own sounds bypass spatialization entirely**, so a weapon fired by the player is centred and full
volume regardless of where the entity is. Without it the player's own footsteps would pan as they turn.

**The pan is a linear function of the dot product with the listener's right vector**, giving a scale of 0 to 2 per
ear. So a sound directly to the right is twice the base volume in that ear and silent in the other — a strong
pan by modern standards, and characteristic of the game's sound.

**Attenuation is linear in distance and reaches zero at a distance of one after scaling**, so the sound's
attenuation parameter determines its audible radius directly. A parameter of zero makes the distance term zero and
the sound audible everywhere at full volume, which is what the protocol's documentation promises
([`protocol.h`](protocol.h.md)).

**There is no elevation component** — only the horizontal right vector is used — so a sound directly above sounds
centred. A rebuild adding elevation changes the game's spatial feel.

## `S_StartSound`

**Contract** — takes an entity, a channel, a sound, a position, a volume and an attenuation; picks a channel,
fills it in, spatializes it, drops it if both ear volumes are zero, and starts it at the current mix position —
skipping forward by however much of the current mix has already been written.

**Invariants** — a new sound starts at `paintedtime`, so it begins at the *front* of the already-mixed region
rather than at the device's read position. That is what keeps sounds in step with each other at the cost of up to
the mix-ahead of latency.

A sound whose spatialization yields silence is dropped rather than occupying a channel.

## `S_StaticSound`

**Contract** — appends a permanently looping sound above the dynamic and ambient channels, spatializes it once,
and never frees it. A count over the channel limit prints and returns.

**Invariants** — static channels are spatialized **once at creation**, not per frame, because neither they nor —
in a static sound's case — the listener's *distance* matters enough. That is why walking past a static sound does
not change its volume, which is audible and is a known limitation.

## `S_StopSound`, `S_StopAllSounds`, `S_StopAllSoundsC`, `S_ClearBuffer`

**Contract** — silence one entity's channel; silence everything and optionally zero the device buffer; the console
command for the same; and write silence over the whole ring.

**Invariants** — the silence value depends on the sample width — zero for sixteen-bit, mid-scale for eight-bit
unsigned — which is a real trap ([`sound.h`](sound.h.md#s_clearbuffer)).

## `S_UpdateAmbientSounds`

**Contract** — reads the four ambient levels from the leaf the listener is in, scales them by the ambient tunable,
and fades each ambient channel's volume toward its target at a configured rate.

```text
FUNCTION s_update_ambient_sounds()
  IF ambient sounds are off OR there is no world  RETURN
  leaf = the leaf containing listener_origin
  IF the leaf is the solid one OR the ambient level is zero
    silence all four ;  RETURN
  FOR EACH of the four ambient channels
    target = the leaf's ambient level for that kind * the ambient tunable
    IF target < 8  target = 0                       # below audibility: silence
    # Fade toward the target rather than jumping.
    IF the current volume < target
      volume = min(volume + frametime * ambient_fade, target)
    ELSE IF the current volume > target
      volume = max(volume - frametime * ambient_fade, target)
    both ear volumes = the faded volume
```

**Invariants** — **the fade is why water becomes audible gradually as you approach it.** The levels are baked per
leaf by the map compiler ([`bspfile.h`](bspfile.h.md)) and change discontinuously as the listener crosses a leaf
boundary; the fade smooths that into a continuous approach.

A target below 8 is snapped to silence, which stops barely-audible ambience from lingering.

## `S_Update`

**Contract** — called once per frame with the listener's position and basis. Records them, updates the ambient
levels, respatializes every dynamic and static channel, drops the silent ones, optionally prints the active
channels, and mixes forward.

## `GetSoundtime`

**Contract** — reads the device's position, counts buffer wraps, and produces an absolute sample-pair position.
Resets everything when the position approaches the integer limit.

```text
FUNCTION get_soundtime()
  fullsamples = the ring's length IN sample pairs
  samplepos = the device's read position
  IF samplepos < oldsamplepos
    buffers = buffers + 1                         # the ring wrapped
    IF paintedtime > 0x40000000
      # Approaching the 32-bit limit: reset everything rather than overflow.
      buffers = 0 ;  paintedtime = fullsamples ;  stop all sounds
  oldsamplepos = samplepos
  soundtime = buffers*fullsamples + samplepos/channels
```

**Invariants** — three things.

**Wraps are counted by noticing the position went backwards**, which means the ring must not wrap twice between
calls. The source's own comment acknowledges this — "oh well" — and a frame taking longer than the whole ring
duration miscounts. At 11 kilohertz with a typical ring that is about a second, so it happens only during a stall.

**The counter is reset near the integer limit**, which at 11 kilohertz is about 27 hours of continuous play. All
sounds are stopped because every channel's end time is in the old numbering.

**A different platform reports an absolute sample count directly** and skips the wrap counting entirely, which is
the better interface — and is what a rebuild should ask of its device.

## `S_Update_`

**Contract** — the mixer's driver: updates the clock, corrects an undershoot, computes the target position as the
device's position plus the mix-ahead bounded by the ring's length, restores a lost device buffer, mixes to the
target, and submits. Does nothing when sound is blocked.

```text
FUNCTION s_update_inner()
  IF sound is not started OR is blocked  RETURN
  get_soundtime()
  IF paintedtime < soundtime  paintedtime = soundtime     # the device overran
                                                          # us: skip ahead
  endtime = soundtime + _snd_mixahead * the device's rate
  bound endtime BY soundtime + the ring's length in pairs
  IF the platform's buffer was lost  restore and restart it
  s_paint_channels(endtime)
  snddma_submit()
```

**Invariants** — **an undershoot is corrected by skipping forward, not by catching up.** If the device consumed
past where the mixer had written, the mixer abandons the gap and resumes at the current position — so a stall
produces a moment of whatever was in the ring rather than a delayed replay. That is the right choice and it is
why the mid-frame top-up calls exist to prevent it
([`sound.h`](sound.h.md#s_extraupdate)).

**The target is bounded by the ring's length**, so the mixer can never write past the device's read position all
the way around.

## `S_ExtraUpdate`

**Contract** — mixes forward without respatializing. Suppressed by a tunable.

**Invariants** — this is what the renderer calls from inside slow passes
([`r_edge.c`](r_edge.c.md#r_scanedges)) and what the file loader calls during a level load. A rebuild with a
callback-driven device deletes it and every call to it.

## `S_LocalSound`, `S_Play`, `S_PlayVol`, `S_SoundList`, `S_SoundInfo_f`

**Contract** — play a named sound at the listener with no attenuation, for interface feedback; the console commands
to play a sound by name and at a volume; and two diagnostics listing the loaded sounds and the device's
negotiated format.

## `S_AmbientOn`, `S_AmbientOff`

**Contract** — enable and disable the ambient channels.

**Invariants** — used around the end-of-level screen, where the listener has no position.
